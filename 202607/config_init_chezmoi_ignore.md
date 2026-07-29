---
tier: tale
title: Stage and host-gate the chezmoi overlay written by sase config init
goal:
  "`sase config init` always materializes the machine overlay in the chezmoi source tree, adds a hostname-guarded
  `.chezmoiignore` stanza for it, and stages both files so the commit/push/apply deployment carries them."
create_time: 2026-07-29 10:39:15
status: done
---

- **PROMPT:** [202607/prompts/config_init_chezmoi_ignore.md](prompts/config_init_chezmoi_ignore.md)
- **AGENTS:**
  - [bbugyi200.athena.o9--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.o9.md#member-code)
  - [bbugyi200.athena.o9--plan](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.o9.md#member-plan)
- **COMMITS:**
  - [9180a1f](https://github.com/sase-org/sase/commit/9180a1fd67ae9b32231153fe65a2378fb179733c) — fix(config):
    initialize chezmoi machine overlays safely

# Plan: Stage and host-gate the chezmoi overlay written by `sase config init`

## Background

`sase config init` (`src/sase/main/config_init_handler.py`) writes the selected machine overlay (`sase_<machine>.yml`).
With `use_chezmoi: true`, `_write_identity_overlay` remaps the target `~/.config/sase/sase_<machine>.yml` to the chezmoi
source path `~/.local/share/chezmoi/home/dot_config/sase/sase_<machine>.yml`, and `_deploy_machine_overlay` hands that
path to `deploy_to_chezmoi` (`src/sase/main/_init_chezmoi_deploy.py`), which does `git add` → `git commit` →
`git pull --rebase` → `git push` → `chezmoi apply --force`.

Two defects were reproduced against the current tree:

### Defect 1 — the chezmoi source file is silently never created (so nothing is staged)

`_write_identity_overlay` reads its "current" text through `_read_overlay_text(write_path, target_path)`, which **falls
back to the applied home target when the chezmoi source file does not exist**. When that applied target already contains
the complete identity, the spliced text equals the fallback text, the function returns `changed=False`, and the handler
skips `_deploy_machine_overlay` entirely. The chezmoi source file is never written, never staged, never committed — and
`sase config init` exits `0` reporting success.

Reproduced in a sandbox HOME (chezmoi source `.../home/dot_config/sase/sase_kellys_mbp.yml` absent, applied
`~/.config/sase/sase_kellys_mbp.yml` already holding `id.username` / `id.machine_name`):

```
Existing machine names: kellys_mbp
Machine name [athena]: kellys_mbp
Selected owner 'bbugyi200' on machine 'kellys_mbp' in /tmp/.../.sase/machine_name.
rc= 0
=== chezmoi source files ===   ->  only sase.yml; no sase_kellys_mbp.yml
=== git log ===                ->  only the initial commit
```

This matches what happened on Kelly's MBP: the overlay ended up committed by hand
(`7cc13658 chore: Add sase_kellys_mbp.yml` in the chezmoi repo) rather than by `sase config init`.

Note for the implementer: the `git add` call in `_deploy_to_chezmoi_locked` itself was verified to work correctly for a
brand-new untracked file when the deploy is actually reached — the gap is that this code path is bypassed, and that **no
existing test exercises real staging** (both chezmoi tests in `tests/main/test_config_init_handler.py` monkeypatch
`deploy_to_chezmoi` away). Fix the materialization gap and add real end-to-end staging coverage; do not "fix" `git add`
itself.

### Defect 2 — the new overlay is applied by chezmoi on every machine

`home/.chezmoiignore` in the chezmoi repo gates each per-machine overlay behind the hostname, e.g. commit `45a993ac5a92`
("fix: Add sase_kellys_mbp.yml to .chezmoiignore") appended:

```
{{ if ne .chezmoi.hostname "Kellys-MBP" }}
.config/sase/sase_kellys_mbp.yml
{{ end }}
```

The full current file is:

```
tags
{{ if ne .chezmoi.fqdnHostname "bbugyi.c.googlers.com" }}
.config/sase/sase_work.yml
{{ end }}
{{ if ne .chezmoi.hostname "athena" }}
.config/sase/sase_athena.yml
{{ end }}
{{ if ne .chezmoi.hostname "Kellys-MBP" }}
.config/sase/sase_kellys_mbp.yml
{{ end }}
```

`sase config init` does not write this stanza, so until the user edits `.chezmoiignore` by hand, a freshly committed
`sase_<machine>.yml` is applied to _every_ machine that runs `chezmoi apply`.

Two facts the implementation must respect:

- The chezmoi hostname is **not** the SASE machine name. Hostname `Kellys-MBP` produces machine name `kellys_mbp`
  (`_hostname_suggestion` lowercases and maps every char outside `[a-z_]` to `_`), and the user may type an entirely
  unrelated machine name at the prompt. The stanza guard must use the real hostname; the ignored path must use the
  overlay filename.
- `.chezmoiignore` lives at `CHEZMOI_HOME/.chezmoiignore` (`~/.local/share/chezmoi/home/.chezmoiignore`) — chezmoi
  special files keep their literal names, so there is no `dot_` encoding. The repo uses a `.chezmoiroot` of `home`,
  which is why `deploy_to_chezmoi` uses `git_root = CHEZMOI_HOME.parent`. The ignored entry is a **target-relative**
  path with real dots: `.config/sase/sase_<machine>.yml`.

## Goal

After this change, running `sase config init` on a new machine with `use_chezmoi: true` leaves the chezmoi repo with
both `home/dot_config/sase/sase_<machine>.yml` and an updated `home/.chezmoiignore` staged and committed in a single
commit, pushed, and applied — with the new overlay applying only on the machine it was initialized on.

## Boundary check (Rust core)

`../sase-core` carries only chezmoi _path layout_ wire types (`crates/sase_core/src/content_layout.rs` →
`ChezmoiContentLayoutWire`). The entire chezmoi source-state deploy pipeline — git staging, commit/push, `chezmoi apply`
— is Python-only in this repo (`src/sase/main/_init_chezmoi_deploy.py`, consumed by `config_init_handler`,
`init_memory_handler`, `init_skills_handler`). This change stays inside that Python CLI pipeline, so no Rust wire/API
change is required. Do not port it to `sase-core`.

## Step 1 — Materialize the chezmoi source overlay even when the applied target already matches

File: `src/sase/main/config_init_handler.py`, `_write_identity_overlay`.

Capture whether the _actual write destination_ already exists before reading, and only take the unchanged short-circuit
when it does:

```python
    write_path = resolve_write_path(str(target_path), use_chezmoi=use_chezmoi)
    if write_path is None:  # pragma: no cover - a concrete target was supplied.
        raise ConfigEditError("machine overlay has no writable destination")
    destination_exists = write_path.exists()
    current_text = _read_overlay_text(write_path, target_path)
    updated_text = set_key(current_text, ("id", "username"), username)
    updated_text = set_key(updated_text, ("id", "machine_name"), machine_name)
    updated_text = unset_key(updated_text, ("machine_name",))
    if destination_exists and updated_text == current_text:
        return write_path, False
    write_path.parent.mkdir(parents=True, exist_ok=True)
    write_path.write_text(updated_text, encoding="utf-8")
    return write_path, True
```

Update the docstring to state that a missing write destination is always materialized (seeding it from the applied
target when one exists) so the chezmoi source tree is never left behind the applied home copy.

Behavior notes:

- `use_chezmoi: false` is unaffected: `write_path == target_path`, so `destination_exists` is exactly
  `target_path.exists()` and the short-circuit still fires for a genuine no-op re-run.
- The `use_chezmoi: true` re-run case is still a no-op once the chezmoi source file exists.

## Step 2 — New helper module for `.chezmoiignore` maintenance

Create `src/sase/main/_init_chezmoi_ignore.py`. Keep it dependency-light (no Textual, no config loading beyond what is
passed in) so it is unit-testable in isolation. Public surface:

### `chezmoi_target_entry(source_path: Path, *, chezmoi_home: Path) -> str | None`

Inverse of `sase.content_layout.chezmoi_source_path`. Take `source_path.relative_to(chezmoi_home)`, map each part
`dot_<rest>` → `.<rest>`, and join with `/`. Return `None` when `source_path` is not under `chezmoi_home` (`ValueError`
from `relative_to`).

Example: `<chezmoi_home>/dot_config/sase/sase_kellys_mbp.yml` → `.config/sase/sase_kellys_mbp.yml`.

Note there is no existing inverse decoder in the tree; this is the first one. Keep it local to this module rather than
adding it to `sase/content_layout.py`, whose helpers are mirrored by the Rust content-layout surface.

### `chezmoi_hostname() -> str | None`

Ask chezmoi itself, because that is what evaluates the template:

1. Run `chezmoi data --format=json` (`subprocess.run(..., capture_output=True, text=True, check=False)`), parse JSON,
   and return `data["chezmoi"]["hostname"]` when it is a non-empty string. Guard `FileNotFoundError`, a non-zero return
   code, `json.JSONDecodeError`, and missing/none keys.
2. Fall back to `socket.gethostname().split(".", 1)[0]` — chezmoi's `hostname` is the short hostname, so the first FQDN
   label is the correct fallback (verified on athena: `chezmoi data` reports `hostname: athena`,
   `fqdnHostname: athena.bbugyi.ddns.net`, and `socket.gethostname()` is `athena`).
3. Return `None` when the result is empty or contains anything outside `[A-Za-z0-9._-]` — a hostname with a quote,
   backslash, or newline would corrupt the Go template, and silently emitting a broken `.chezmoiignore` is worse than
   skipping.

Allow tests to inject the subprocess runner (e.g. a module-level `_run_chezmoi_data` seam that tests monkeypatch) so no
test shells out.

### `ensure_machine_ignore_entry(*, chezmoi_home: Path, entry: str, hostname: str) -> Path | None`

Idempotently append the guarded stanza to `chezmoi_home/.chezmoiignore`.

- Read the file when it exists, else start from `""`.
- If any line, after stripping surrounding whitespace, equals `entry`, return `None` (already gated — leave the existing
  guard alone, whether it uses `.chezmoi.hostname` or `.chezmoi.fqdnHostname`). Exact-line matching is required so the
  `sase_work.yml` fqdn stanza and other machines' stanzas are neither matched nor rewritten.
- Otherwise normalize the existing text to end with exactly one `\n` (no-op for empty text) and append, matching the
  format of commit `45a993ac5a92` exactly — no blank separator line:

  ```
  {{ if ne .chezmoi.hostname "<hostname>" }}
  <entry>
  {{ end }}
  ```

- `mkdir(parents=True, exist_ok=True)` the parent, write with `encoding="utf-8"`, and return the `.chezmoiignore` path.

Export the three functions in `__all__`.

## Step 3 — Wire the ignore stanza into `sase config init`

File: `src/sase/main/config_init_handler.py`.

Change `_deploy_machine_overlay` to accept an iterable of paths instead of one path, and pass that iterable to both
`defer_chezmoi_paths` and `deploy_to_chezmoi` (the `ChezmoiDeployBehavior` is unchanged, including the
`chore: initialize SASE owner identity` commit message):

```python
def _deploy_machine_overlay(
    paths: Sequence[Path], args: argparse.Namespace
) -> int:
    if defer_chezmoi_paths(paths, chezmoi_home=config_core.CHEZMOI_HOME):
        return 0
    return deploy_to_chezmoi(paths, ChezmoiDeployBehavior(...))
```

Add a private helper that produces the ignore path to deploy, returning `None` when there is nothing to add:

```python
def _ensure_machine_overlay_ignored(overlay_write_path: Path) -> Path | None:
```

- Compute `entry = chezmoi_target_entry(overlay_write_path, chezmoi_home=config_core.CHEZMOI_HOME)`. Return `None` when
  it is `None` (write path not under the chezmoi source tree).
- Resolve `hostname = chezmoi_hostname()`. When it is `None`, print a warning to `stderr` naming the file that will
  otherwise apply everywhere, and return `None`:
  `config init: could not determine the chezmoi hostname; add a `.chezmoiignore` guard for <entry> manually`.
- Return `ensure_machine_ignore_entry(chezmoi_home=..., entry=entry, hostname=hostname)`.

In `run_config_init`, replace the tail:

```python
    if use_chezmoi and overlay_changed and overlay_write_path is not None:
        deploy_paths: list[Path] = [overlay_write_path]
        try:
            ignore_path = _ensure_machine_overlay_ignored(overlay_write_path)
        except OSError as exc:
            print(
                f"error: failed to update the chezmoi ignore list: {exc}",
                file=sys.stderr,
            )
            return 1
        if ignore_path is not None:
            print(f"Restricted {machine_name} overlay to this machine in {ignore_path}")
            deploy_paths.append(ignore_path)
        return _deploy_machine_overlay(deploy_paths, args)
    return 0
```

Requirements this ordering satisfies, and which the implementer must preserve:

- The `.chezmoiignore` edit happens **before** the single `deploy_to_chezmoi` call, so both files land in the same
  `git add` → `git commit` → `git push`, and the trailing `chezmoi apply --force` already sees the final ignore rules.
- Under bare `sase init`, `defer_chezmoi_paths` receives both paths, so the deferred consolidated deploy stages both
  too.
- An ignore-write `OSError` is fatal (`return 1`): committing the overlay without its guard is the exact failure mode
  this plan exists to prevent. An _undeterminable hostname_ is not fatal — it warns and still deploys the overlay, since
  the identity write is the primary user-visible outcome.
- `use_chezmoi: false` never touches `.chezmoiignore`.

Import `chezmoi_hostname`, `chezmoi_target_entry`, and `ensure_machine_ignore_entry` from `._init_chezmoi_ignore` at
module top (alongside the existing `._init_chezmoi_deploy` import), and `Sequence` from `collections.abc`.

## Step 4 — Tests

### `tests/main/test_config_init_handler.py` (update existing)

`test_new_chezmoi_overlay_uses_direct_deploy` and `test_new_chezmoi_overlay_joins_deferred_bare_init_deploy` monkeypatch
`resolve_write_path` to a `tmp_path/chezmoi/home/dot_config/sase/sase_athena.yml` that is not under the real
`CHEZMOI_HOME`, so `chezmoi_target_entry` returns `None` and no ignore path is added. Confirm they still assert
`deployed == [source]` / `deferred.paths == [source]` and adjust only if the new signature requires it
(`_deploy_machine_overlay` now receives a list).

### `tests/main/test_config_init_chezmoi.py` (new)

Use the existing `_prepare` / `_args` helpers' shape from `test_config_init_handler.py`
(`monkeypatch.setattr(config_core, "CONFIG_DIR", ...)`, `get_use_chezmoi`, stubbed `_machine_hood_collisions`,
`_TtyStringIO`). Monkeypatch `config_core.CHEZMOI_HOME` to `tmp_path/chezmoi/home`, monkeypatch
`config_init_handler.resolve_write_path` to remap into that tree, and monkeypatch the module's hostname seam to a fixed
value such as `Kellys-MBP`.

1. **Regression for Defect 1** — applied target `<config_dir>/sase_kellys_mbp.yml` already holds the complete identity
   while the chezmoi source file is absent. Assert the chezmoi source file is created with the same content and that the
   deploy is invoked with it. This test fails on the current tree.
2. **Real staging, end to end** — build an actual git repo at `tmp_path/chezmoi` (`git init`, a seed commit,
   `user.email`/`user.name` set) with `CHEZMOI_HOME = tmp_path/chezmoi/home`, do **not** monkeypatch
   `deploy_to_chezmoi`, and monkeypatch `subprocess.run` for `chezmoi apply` (or set the behavior's `no_apply` path) so
   no real chezmoi runs. After `run_config_init`, assert `git ls-files` in that repo contains both
   `home/dot_config/sase/sase_kellys_mbp.yml` and `home/.chezmoiignore`, and that `git show --stat HEAD` names both.
   There is no upstream, so `skip_pull_push_without_upstream` short-circuits pull/push — that is fine and worth
   asserting is not an error (`rc == 0`).
3. **`.chezmoiignore` created when absent** — file did not exist; assert its exact content is the three-line stanza for
   `.config/sase/sase_kellys_mbp.yml` guarded by `Kellys-MBP`.
4. **Appends without disturbing existing stanzas** — seed `.chezmoiignore` with the real-world content quoted in
   Background (the `tags` line, the `fqdnHostname` `sase_work.yml` stanza, and the `athena` stanza). Assert the seeded
   text is preserved verbatim as a prefix and the new stanza is appended.
5. **Idempotent** — run twice (or seed a file that already contains the entry under a _different_ guard form) and assert
   `.chezmoiignore` is byte-identical and no duplicate entry is added.
6. **Deferred bare `sase init`** — inside `defer_chezmoi_deploy()`, assert `deferred.paths` contains both the overlay
   source path and the `.chezmoiignore` path.
7. **`use_chezmoi: false`** — assert `.chezmoiignore` is never created and no deploy is attempted.
8. **Missing trailing newline** — seed `.chezmoiignore` with content not ending in `\n`; assert the result is
   well-formed (single newline before the appended stanza).
9. **Unknown hostname** — hostname seam returns `None`; assert a warning goes to stderr, no `.chezmoiignore` is written,
   and the overlay is still deployed with `rc == 0`.

### `tests/main/test_init_chezmoi_ignore.py` (new, unit)

- `chezmoi_target_entry` decodes `dot_` parts, handles nested dot dirs, and returns `None` for a path outside
  `chezmoi_home`.
- `chezmoi_hostname` prefers the `chezmoi data` JSON value, falls back to the first label of `socket.gethostname()` when
  the command is missing / fails / returns unparseable JSON, and returns `None` for an empty or invalid-character
  hostname.
- `ensure_machine_ignore_entry` returns `None` and leaves the file untouched when the entry is already present, and
  returns the path after appending otherwise.

## Step 5 — Documentation

`docs/configuration.md`, in the paragraph ending "Direct `sase config init` uses the normal commit/push/apply
deployment; bare `sase init` combines the edit with deferred chezmoi deployment." (around lines 102-106): add that with
`use_chezmoi: true`, `sase config init` also appends a hostname-guarded stanza to the chezmoi source `.chezmoiignore` so
the new `sase_<machine>.yml` is applied only on the machine it was initialized on, and that both files are staged in the
same commit. Show the stanza shape. Mention that the guard uses the chezmoi hostname (which may differ from the SASE
machine name) and that an existing entry for the file is left alone.

No new CLI flags or subcommands are added, so `sase/memory/cli_rules.md` does not apply and no `docs/init.md`
command-table row changes.

## Verification

```bash
just install
just lint
just test
just check
```

Targeted runs while iterating:

```bash
.venv/bin/python -m pytest tests/main/test_config_init_handler.py \
  tests/main/test_config_init_chezmoi.py tests/main/test_init_chezmoi_ignore.py -q
```

Manual sandbox check (mirrors the reproduction above; run from outside the repo root so the workspace's `sase/`
directory does not shadow the package, and use the workspace venv interpreter):

1. Build a fake `HOME` with `.config/sase/sase.yml` containing `use_chezmoi: true` and a git repo at
   `$HOME/.local/share/chezmoi` holding `home/dot_config/sase/sase.yml` plus `home/.chezmoiignore`.
2. Pre-create `$HOME/.config/sase/sase_kellys_mbp.yml` with a complete `id:` block (the Defect 1 trigger).
3. Call `run_config_init` with an `argparse.Namespace` carrying `check=False`, `no_apply=True`, a fake `_init_stdin`
   whose `isatty()` returns `True`, and an `_init_input_func` feeding the prompts.
4. Assert the chezmoi repo's HEAD commit contains both `home/dot_config/sase/sase_kellys_mbp.yml` and
   `home/.chezmoiignore`, and that `.chezmoiignore` gained exactly one guarded stanza.

## Out of scope

- Changing `deploy_to_chezmoi`'s staging, commit, pull/push, or locking behavior.
- Retrofitting `.chezmoiignore` management onto `sase memory init` or `sase skill init`, whose generated files are
  intentionally machine-agnostic.
- Backfilling `.chezmoiignore` stanzas for overlays committed before this change.
- Reconciling a machine overlay whose recorded hostname later changes.
