---
tier: tale
title: Fix sase-listen render EXDEV at the master stage
goal:
  sase-listen render masters and commits episodes even when the system temp dir is on a
  different filesystem (tmpfs /tmp on athena), guarded by regression tests, and athena's
  installed renderer carries the fix so the user's Symphony render succeeds on retry.
size: small
proposed_by: bbugyi200.athena.0ws.f0
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.0ws.f0](https://github.com/sase-org/sase--agents/blob/main/sessions/bbugyi200.athena.0ws.f0.md)
- **COMMITS:**
  - [870e069](https://github.com/sase-org/sase-listen/commit/870e0691e12a1fcdf151ae8fd9fd45d1d4143050)
    — fix(audio): master the MP3 beside its target so a tmpfs /tmp cannot break the
    replace

# Plan: Fix sase-listen `render` dying with EXDEV at `stage: master`

## Symptom

On athena, from an interactive shell:

```
❯ sase-listen render https://openai.com/index/open-source-codex-orchestration-symphony/ -e full
stage: synthesize
[1/10] chunk 0 synthesized … [10/10]
stage: gates
stage: master
sase-listen render: unexpected error: [Errno 18] Invalid cross-device link:
  '/tmp/sase-listen-master-io0rm4x_/episode.mp3' ->
  '/home/bryan/.local/share/sase-listen/library/.staging/an-open-source-spec-for-codex-orchestration-symphony-808675/master.mp3'
```

## Root cause (confirmed)

`master_to_mp3()` in `src/sase_listen/audio/mastering.py` (sase-listen repo) does all of
its ffmpeg work inside `tempfile.TemporaryDirectory(prefix="sase-listen-master-")`, i.e.
under the system temp dir. The pass-2 MP3 is encoded to `<tmp>/episode.mp3`, then moved
onto the caller's `out_path` with `os.replace(staging, out)`. `os.replace` is
`rename(2)`, which cannot cross filesystems. The pipeline (`pipeline.py`, master stage)
passes `staging_path(episode_id, library_root) / "master.mp3"`, which lives under
`$XDG_DATA_HOME/sase-listen/library/.staging/`.

- athena: `/tmp` is **tmpfs** (`findmnt -T /tmp` → `tmpfs`), the library is on ext4
  `/dev/nvme1n1p2` → different devices → `EXDEV` every time `TMPDIR` is unset or `/tmp`.
- apollo: `/tmp` and `$HOME` share the same ext4 root, so the original apollo rollout
  never hit it.
- The 2026-10-05 athena proof episode (`harness-engineering…`) succeeded only because it
  was rendered by a SASE agent, whose `TMPDIR` points under `~/.cache/sase/tmp/…` (same
  ext4 as the library).
- The test suite cannot see it: pytest's `tmp_path` and the mastering temp dir both live
  under the same system temp root.

Reproduced offline (tone narrator, no API spend, scratch XDG dirs) with the installed
binary: `TMPDIR=/tmp` → the identical EXDEV; the same command with `TMPDIR` on the same
ext4 disk succeeds.

Audit of every other temp→final move in `src/`: all other `os.replace` calls (`feed.py`,
`feedhost.py`, `writer/author.py`, `web/store.py`, `library.py`, `cache.py`) use a temp
file created beside its target, and `pipeline.py`'s `-o` copy uses `shutil.copyfile` +
`shutil.move` (cross-device safe). `master_to_mp3` is the only offender.

## Fix

All work happens in the **sase-listen** linked repo. Open it with
`sase repo open sase-listen -r "<reason>"`, work only in the printed path, and read its
`AGENTS.md` first. sase-listen must not import `sase`.

### 1. `src/sase_listen/audio/mastering.py` — encode beside the target

Keep the WAV and pass 1 in the system temp dir (fast on tmpfs, never renamed). Change
only where pass 2 writes: create the MP3 temp file in `out.parent` so the final
`os.replace` is a same-directory, same-filesystem rename. Follow the existing
`cache.py::_atomic_write_bytes` idiom (`mkstemp(dir=…, prefix=".tmp-")` plus unlink on
`BaseException`). Sketch:

```python
        # Encode beside the target so the final replace never crosses
        # filesystems: the system temp dir is often tmpfs while the library
        # is not, and rename(2) fails with EXDEV across devices.
        fd, staging = tempfile.mkstemp(dir=out.parent, prefix=".tmp-", suffix=".mp3")
        os.close(fd)
        try:
            pass2 = _run_ffmpeg([... same args ..., staging], "loudness normalization pass")
            applied = _parse_loudnorm_json(pass2.stderr)
            os.replace(staging, out)
        except BaseException:
            with contextlib.suppress(OSError):
                os.unlink(staging)
            raise
```

Notes:

- Keep the `.mp3` suffix: ffmpeg picks the muxer from the output extension (`-y` already
  overwrites the empty mkstemp file).
- `out.parent` must already exist; that was already required by the old `os.replace`,
  and the pipeline creates it (`staging.parent.mkdir(...)`).
- Add `import contextlib` if not present. Keep `mypy --strict` clean.
- Update the docstring's "with atomic replace" wording to say the MP3 is staged beside
  `out_path` so the replace stays on one filesystem.

### 2. Regression tests (must fail before step 1, pass after)

Create `tests/conftest.py` with a fixture, e.g. `tmp_on_separate_fs`, that emulates a
tmpfs system temp dir:

- make `tmp_path / "system-tmp"`, and
  `monkeypatch.setattr(tempfile, "tempdir", str(system_tmp))`;
- wrap `os.replace` and `os.rename` (monkeypatch on the `os` module) so a call raises
  `OSError(errno.EXDEV, "Invalid cross-device link")` when **exactly one** of
  `src`/`dst` resolves under `system_tmp`; otherwise delegate to the real function. (Do
  not use "different parent dir" as the condition: `library.atomic_commit` legitimately
  renames `.staging/<id>/x` → `<id>/x`.) `shutil.move` already falls back to copy on
  `OSError`, matching reality.
- yield `system_tmp`.

Tests:

- `tests/audio/test_mastering.py`:
  - `master_to_mp3` into `tmp_path / "library" / "episode.mp3"` under the fixture
    succeeds, the file is a real MP3 (`> 10_000` bytes), and `library/` contains only
    `episode.mp3` (no `.tmp-*` leftovers).
  - Failure cleanup + atomicity: pre-create `out` with `b"old"`, monkeypatch
    `mastering._run_ffmpeg` to delegate the measurement pass but raise `SaseListenError`
    for the `"loudness normalization pass"`; assert the error propagates, `out` still
    reads `b"old"`, and no `.tmp-*` file remains in `out.parent`.
- `tests/test_pipeline.py`: an end-to-end tone render under both `isolated` and the new
  fixture (reuse `_render_tiny(tmp_path)`), asserting it returns a `RenderResult` and
  the episode MP3 exists in the library. This guards the whole render path, not just
  mastering.

Run the new tests before the fix to confirm they reproduce the EXDEV failure, then after
the fix to confirm they pass.

### 3. Docs

- `docs/reliability.md`: the bullet saying "mastering writes through a temp file plus
  atomic replace" → say the MP3 is encoded to a temp file beside the target and
  atomically replaced, so a tmpfs `/tmp` cannot break the rename.
- `docs/troubleshooting.md`: add a short section "`Invalid cross-device link` at
  `stage: master`": the cause, that releases with this fix are immune, how to upgrade,
  and that synthesized chunks and article scripts are cached, so re-running the same
  `render` command after upgrading costs no TTS or writer calls.

Do not edit `CHANGELOG.md` by hand (release-please owns it); the conventional commit
subject should be like
`fix(audio): master the MP3 beside its target so a tmpfs /tmp cannot break the replace`.

## Verification

1. `sase tool run check` in the sase-listen checkout (lint + mypy strict + codespell +
   pytest) passes.
2. Live offline repro on athena (confirm `findmnt -T /tmp` reports `tmpfs`), with the
   checkout's venv binary, then with the reinstalled tool (step 3 below). No API calls,
   nothing published, user library untouched:

   ```bash
   S=$(mktemp -d -p ~/.cache sase-listen-exdev-XXXX)
   printf '# Probe\n\n## One\n\nHello there, this is a short probe paragraph for mastering.\n' > "$S/probe.md"
   TMPDIR=/tmp XDG_DATA_HOME="$S/data" XDG_CACHE_HOME="$S/cache" XDG_STATE_HOME="$S/state" \
     <sase-listen binary> render "$S/probe.md" -n tone --no-publish
   rm -rf "$S"
   ```

   Before the fix this ends with `[Errno 18] Invalid cross-device link`; after it, it
   prints `Duration … LUFS: -16.0`.

3. Deploy to athena's renderer so the user can retry immediately. athena's `uv tool`
   install is a non-editable directory install from a SASE workspace checkout. After
   `check` passes, reinstall from the sase-listen checkout path printed by
   `sase repo open`: `uv tool install --force --reinstall <that path>`, then rerun the
   step-2 probe with the installed `sase-listen` and run `sase-listen doctor`. Once the
   commit is on `origin/master`,
   `uv tool install --force --reinstall git+https://github.com/sase-org/sase-listen` is
   equivalent.

## Out of scope

- Do not run the real Symphony render yourself (it auto-publishes to the user's feed).
  Tell the user to rerun
  `sase-listen render https://openai.com/index/open-source-codex-orchestration-symphony/ -e full`.
  Its full narration script and all 10 chunks are already cached, so it should go
  straight to master/tag/commit/publish without paid calls. The empty leftover
  `.staging/an-open-source-spec-…-808675/` dir is removed by `atomic_commit` on that
  run.
- apollo (feed host) is unaffected (`/tmp` on its root ext4). Upgrading it is optional.
- No changes to the sase repo or to the earlier research report.
