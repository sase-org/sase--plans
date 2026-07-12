---
create_time: 2026-07-09 14:23:54
status: done
prompt: 202607/prompts/vcs_log_default_tags_fetch_ttl.md
tier: tale
---
# Plan: default VCS log tags and throttle remote fetches

## Goal

Make `sase vcs log` show SASE commit tags by default, invert the tag flag to `-T/--no-tags`, and reduce repeated command
latency by skipping remote fetches when the relevant repo/ref was fetched successfully within the last 60 seconds.

This is a single-repo `sase` change. It should not require Rust core or provider plugin API changes because the tag
parsing/rendering and the existing `fetch_remote(cwd, refs)` hook already provide the needed extension points.

## User-facing CLI behavior

1. Tags become on by default for every `sase vcs log` format.
   - `pretty` and `full` render the styled tag chips/rows by default.
   - `oneline` includes the current plain tag suffix by default.
   - `json` includes the `sase_tags` object by default.
2. Add `-T, --no-tags` to hide tags in all formats.
3. Replace the visible `-t, --tags` help surface with `-T, --no-tags`.
   - Recommended compatibility detail: keep `-t/--tags` as a hidden no-op alias for one release so existing saved
     commands still parse, but do not show it in `-h|--help` output or docs.
4. Keep `-N, --no-fetch` as the hard "do not touch remotes" option.
5. Add `-F, --fetch` to explicitly fetch remote refs now, bypassing the 60-second freshness check.
6. Treat `--fetch` and `--no-fetch` as mutually exclusive and fail with parser exit code 2 if both are passed.

The `sase vcs log -h` option list should remain sorted by long option name: `--author`, `--branch/--ref`, `--color`,
`--current-only`, `--fetch`, `--format`, `--limit`, `--no-fetch`, `--no-tags`, `--repo`, `--reverse`, `--since/--after`,
`--until/--before`.

## Fetch freshness model

Add a small VCS-log fetch cache helper, for example `src/sase/vcs_log/fetch_cache.py`.

The cache records successful fetch times keyed by the checkout and remote ref:

- checkout identity: `Path(repo.path).expanduser().resolve(strict=False)`
- remote ref: the resolved ref passed to `fetch_remote`, such as `origin/main`
- timestamp: wall-clock epoch seconds from `time.time()`

Use per-checkout keys rather than a global timestamp. A `git fetch` updates the remote-tracking refs in one checkout's
`.git` directory, so a fetch in one repo does not prove another repo, another linked checkout, or another remote ref is
fresh.

Default mode becomes "auto fetch":

1. Resolve the repo's remote comparison ref as today.
2. If `--no-fetch` is set, skip fetching and use existing remote-tracking refs.
3. If `--fetch` is set, call `provider.fetch_remote(...)` regardless of cache.
4. Otherwise, read the cache.
   - If the same checkout/ref has a successful fetch timestamp with `0 <= now - fetched_at < 60`, skip the fetch.
   - If no fresh timestamp exists, call `provider.fetch_remote(...)`.
5. On fetch success, update the cache timestamp.
6. On fetch failure, preserve today's warning behavior and do not mark the ref fresh. The cache represents freshness of
   local remote-tracking refs, not just "we tried a network operation".

Cache writes should be best-effort and atomic (`tmp` file plus `os.replace`). Missing, corrupt, schema-mismatched, or
unwritable cache files should never make `sase vcs log` fail. They should degrade to "fetch when needed". To keep the
file bounded, prune entries older than a modest retention window, such as 7 days, when writing.

Suggested cache path:

```text
sase_home() / "vcs_log_fetch_cache.json"
```

Suggested JSON envelope:

```json
{
  "schema_version": 1,
  "entries": {
    "<stable-key>": {
      "repo_path": "/expanded/path",
      "remote_ref": "origin/main",
      "fetched_at": 1783620000.0
    }
  }
}
```

The stable key can be a hash of `repo_path + "\0" + remote_ref` to avoid awkward JSON object keys while still storing
readable metadata.

## Data model and rendering notes

The existing `RepoRemoteState.fetched` field currently answers "did this run fetch?" well enough for the old
implementation, but throttling introduces a third useful state: "not fetched this run because it was fetched recently".

Recommended model update:

- Add `fetched_at: float | None = None` to `RepoRemoteState`.
- Keep `fetched` for backward compatibility as "this invocation performed a fetch successfully".
- When the cache skips a fetch, set `fetched=False` and `fetched_at` to the cached timestamp.

Rendering can then avoid misleading "not fetched" text:

- all repos fetched in this invocation: `vs origin/main · fetched`
- all repos fresh from cache: `vs origin/main · fetched <age> ago`
- mixed fetched/cache states: `vs origin/main · fresh`
- explicit `--no-fetch`: keep today's `not fetched`

JSON should keep the existing `fetched` field and add `fetched_at` so consumers can distinguish fresh-from-cache from
fetched-now. `--no-tags` should omit the `sase_tags` field, preserving the current opt-in shape when tags are disabled.

## Implementation steps

1. Update `src/sase/main/parser_vcs.py`.
   - Change the tag option to `-T/--no-tags`, `dest="show_tags"`, `action="store_false"`, defaulting `show_tags=True`.
   - Add hidden `-t/--tags` compatibility alias, if accepted during review.
   - Add `-F/--fetch`, `action="store_true"`, and make it mutually exclusive with `-N/--no-fetch`.
   - Keep help text sorted and clear.
2. Update `src/sase/main/vcs_handler.py`.
   - Pass `show_tags=args.show_tags` into the renderer.
   - Pass the new fetch intent into `run_vcs_log`, preferably as an explicit fetch mode or a `force_fetch` boolean
     alongside existing `no_fetch`.
3. Add the fetch cache helper under `src/sase/vcs_log/`.
   - Pure functions for read, write, freshness check, key generation, and success recording.
   - Inject `now` and optional cache path in tests.
4. Update `src/sase/vcs_log/collect.py`.
   - Accept the new fetch mode.
   - Before `provider.fetch_remote(...)`, consult the cache unless no-fetch or force-fetch mode applies.
   - Record cache timestamps only after successful fetches.
   - Preserve per-repo failure isolation and current warning strings for failed fetches.
5. Update `src/sase/vcs_log/models.py` and `src/sase/vcs_log/render.py`.
   - Add `fetched_at` to `RepoRemoteState`.
   - Render a fresh cached fetch distinctly from explicit no-fetch.
   - Include `fetched_at` in JSON repo entries.
6. Update tests.
   - Parser defaults and aliases in `tests/main/test_vcs_parser.py`.
   - Handler namespace fixtures that currently pass `tags=False`.
   - Tag rendering defaults and `--no-tags` suppression in `tests/test_vcs_log_render.py`.
   - Fetch TTL behavior in `tests/test_vcs_log_collect.py` and focused cache helper tests:
     - first run fetches and records
     - second run within 60 seconds skips
     - run after 60 seconds fetches again
     - `--fetch` bypasses a fresh timestamp
     - `--no-fetch` never fetches and never records
     - different repo paths and different refs are independent
     - failed fetches are not recorded
     - corrupt cache is ignored
7. Update any CLI help snapshots or docs that enumerate `sase vcs log` options.

## Verification

After implementation:

```bash
just install
.venv/bin/python -m pytest \
  tests/main/test_vcs_parser.py \
  tests/test_vcs_log_collect.py \
  tests/test_vcs_log_render.py \
  tests/test_vcs_log_tags.py
just check
```

Also run CLI smoke checks with color disabled:

```bash
sase vcs log --color never --limit 3
sase vcs log --color never --limit 3 --no-tags
sase vcs log --color never --limit 3 --no-fetch
sase vcs log --color never --limit 3 --fetch
```

Expected risks:

- Default JSON output will now include `sase_tags`; this is intentional but is the only machine-readable surface change.
- Concurrent `sase vcs log` runs can still both fetch before either writes the cache. That is acceptable because fetch
  is safe and the cache is only a latency optimization.
