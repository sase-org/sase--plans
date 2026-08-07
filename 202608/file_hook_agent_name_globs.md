---
tier: tale
title: file_hooks agent_name_globs filter + globs -> path_globs rename
goal:
  A file hook can filter events by the SASE agent name that produced them via a new
  `agent_name_globs` field (with `!` negation), the existing `globs` field is renamed to
  `path_globs` with a loud fail-soft error on the legacy key, and the chezmoi-managed
  `research-highlights` hook uses both fields so it fires on more research markdown
  while ignoring the two pre-consolidation `#research_swarm` researchers.
proposed_by: bbugyi200.athena.v8
create_time: 2026-08-07 19:21:18
status: wip
---

# Plan: `file_hooks` `agent_name_globs` filter + `globs` → `path_globs` rename

## Problem

The `research-highlights` file hook in the chezmoi-managed global `sase.yml` currently
filters only on repo-relative path:

```yaml
file_hooks:
  - name: research-highlights
    description:
      Render new consolidated research reports into Highlights PDFs for the Obsidian
      reading queue.
    command: bob highlights create
    sidecars: [research]
    globs: ["20*/*/*.md", "!20*/*/*__*.md"]
    ops: [ADD]
    timeout: 120s
```

`20*/*/*.md` is exactly three path segments, so it only ever matches the consolidated
report the `#research_swarm` lead writes at `<YYYYmm>/<name>/<name>.md`. Research
reports that live directly in a month directory (`<YYYYmm>/<name>.md`) — including
reports from a plain `#research` agent — never get a Highlights PDF.

Broadening the path filter to `20*/**/*.md` fixes that, but it also starts matching the
two independent researchers' pre-consolidation reports. In `#research_swarm` (see
`sase/repos/linked/chezmoi/home/sase/xprompts/research_swarm.md`) the clan is
`research.<N>` with members `research.<N>.cdx`, `research.<N>.cld`,
`research.<N>.final`, and `research.<N>.image`. The `.cdx` and `.cld` researchers each
write a report into the month directory, and only later does `.final` move them to
`<name>/<name>__a.md` and `<name>/<name>__b.md` and write the real consolidated report.
Path globs alone cannot tell those throwaway reports apart from a report worth
rendering, because at the moment they are committed they look exactly like a normal
top-level research report.

The missing dimension is _who produced the file_. There is no way to express "run this
hook only for files a particular set of SASE agents did (or did not) write."

Separately, `globs` is ambiguous once a second glob-shaped filter exists.

## Goal

1. Rename the `file_hooks` `globs` field to `path_globs`.
2. Add an `agent_name_globs` filter that matches the SASE agent name attributed to the
   file event, supporting `!`-prefixed negation exactly like `path_globs`.
3. Update the chezmoi `research-highlights` hook to:

   ```yaml
   path_globs: ["20*/**/*.md", "!20*/*/*__*.md"]
   agent_name_globs: ["!research.*.cld", "!research.*.cdx"]
   ```

## Background: how a file-hook event is produced today

Everything below is Python-only. There is no `file_hook` code in
`../sase-core/crates/sase_core` (verified by grep), so this change stays entirely in
this repo and introduces no new crossing of the Rust core backend boundary.

- `src/sase/config/file_hooks.py` owns `FileHookConfig` (the validated config entry),
  `FileHookEvent` (the small matcher view of one file event), `_parse_file_hook`,
  `_glob_matches`, and `hook_matches_event`.
- `src/sase/file_hooks/engine.py` owns `CapturedFileEvent` (the full attributed event),
  the two producer seams, and the detached batch payload:
  - `emit_commit_file_hook_events()` → `_derive_commit_file_events()` — called from
    `src/sase/workflows/commit/workflow.py:286` (`sase commit`) and
    `src/sase/sdd/_commit_store.py:153` (`_emit_sdd_file_hooks`, the sidecar commit path
    the research sidecar uses).
  - `capture_artifact_file_event()` / `emit_artifact_file_hook_event()` — called from
    `src/sase/artifact_cli/create.py`.
- `src/sase/file_hooks/runner.py` executes a claimed batch and sends one notification
  per run.
- `src/sase/main/file_hook_handler.py` renders `sase file-hook list`.

Both producer seams run **in the agent's own process** during `sase commit` /
`sase artifact create`, so the agent identity is available from the environment at
capture time. `src/sase/workflows/commit/runtime_tags.py:63` already resolves it
correctly for commit provenance tags: read `agent_meta.json`'s `name` from
`$SASE_ARTIFACTS_DIR` first, then fall back to `$SASE_AGENT_NAME`. Metadata must win
because a family member can replace another inside one process, leaving
`SASE_AGENT_NAME` pointing at the lane rather than the concrete member.

## Verified matching semantics

`_glob_matches` uses `wcmatch.glob.globmatch` with
`DOTGLOB | GLOBSTAR | NEGATE | NEGATEALL`. That flag set was exercised against the exact
target patterns in this workspace, and the results below are measured, not assumed:

`path_globs: ["20*/**/*.md", "!20*/*/*__*.md"]`

| repo-relative path     | matches | note                                        |
| ---------------------- | ------- | ------------------------------------------- |
| `202608/foo.md`        | `True`  | newly matched (was `False` before)          |
| `202608/foo/foo.md`    | `True`  | the consolidated report; matched before too |
| `202608/foo/foo__a.md` | `False` | vetoed by the negative                      |
| `202608/foo__a.md`     | `True`  | see "known consequence" below               |
| `202608/a/b/c.md`      | `True`  | newly matched                               |

`**` matches zero or more segments under GLOBSTAR, which is why `20*/**/*.md` still
covers the plain `202608/foo.md` case.

`agent_name_globs: ["!research.*.cld", "!research.*.cdx"]` (agent names contain no `/`,
so `*` never has a separator to refuse to cross):

| agent name             | matches |
| ---------------------- | ------- |
| `research.7.cld`       | `False` |
| `research.7.cdx`       | `False` |
| `research.7.final`     | `True`  |
| `research.7.image`     | `True`  |
| `bbugyi200.athena.cld` | `True`  |
| `""` (unattributed)    | `True`  |

And with a list that contains at least one positive pattern (`["research.*"]` or
`["research.*", "!research.*.cld"]`), the unattributed `""` case measures `False`.

That is exactly the desired rule, and it falls out of NEGATE/NEGATEALL for free:

- **Negative-only list** → "every agent except these", and an event with no resolvable
  agent name still runs the hook.
- **Any positive pattern present** → the event must be attributed to a matching agent,
  so an unattributed event does not run the hook.

Implementation consequence: represent "no agent name" as the empty string when matching
(`event.agent_name or ""`), and document that rule.

### Known consequence of the requested path globs

`202608/foo__a.md` (a `__`-suffixed file sitting directly in a month directory) matches,
because the `!20*/*/*__*.md` veto requires exactly one intermediate directory. In
practice those files only exist between the researchers writing them and `.final` moving
them into `<name>/`, and both researchers are excluded by `agent_name_globs`, so this is
harmless. Do not "fix" it by editing the requested globs — they are the user's explicit
spec.

## Implementation

### 1. Share the agent-name resolver — `src/sase/agent/identity.py`

`src/sase/agent/identity.py` is already the documented home for "Shared SASE agent
identity discovery helpers", but the metadata-first resolver lives in the commit
workflow. `file_hooks` should not import identity from the commit workflow.

- Add `resolve_local_agent_name(env: Mapping[str, str] | None = None) -> str | None` to
  `src/sase/agent/identity.py`. It reads `agent_meta.json` from `SASE_ARTIFACTS_DIR` and
  returns its cleaned `name` when present, otherwise the cleaned `SASE_AGENT_NAME`.
  Reuse the existing `_agent_meta_from_dir` / `_clean_value` helpers. Add it to
  `__all__`.
  - Note this is deliberately **not** `discover_agent_identity()`, which is env-first;
    metadata-first ordering is required for family members (see the comment at
    `runtime_tags.py:76-80`, and carry that reasoning into the new docstring).
  - Only the `name` key is consulted here, unlike `agent_name_from_meta`, which also
    accepts `workflow_name`/`agent_name`. Keep the narrow `name`-only behavior so commit
    provenance is unchanged.
- Rewrite `src/sase/workflows/commit/runtime_tags.py:63` `resolve_local_agent_name()` as
  a thin delegation that preserves its current tag sanitization:
  `return _sanitize_tag_value(identity_resolve_local_agent_name())`. Keep the existing
  public name and the module's exported behavior identical — commit footers must not
  change.

### 2. Config — `src/sase/config/file_hooks.py`

- `FileHookConfig`: rename the `globs` field to `path_globs`, and add
  `agent_name_globs: tuple[str, ...] | None` immediately after it. Field order should
  read
  `projects, sidecars, path_globs, agent_name_globs, ops, timeout_seconds, source_layer`.
- `FileHookEvent`: add `agent_name: str | None = None` as the final field. Do not
  validate it in `__post_init__`; `None` is legitimate for a non-agent commit. The
  dataclass stays frozen and therefore hashable, which `_batch_payload`'s
  `captured_by_match` dict relies on.
- `_parse_file_hook`:
  - read `path_globs` and `agent_name_globs` through the existing
    `_optional_string_tuple`.
  - **Reject unknown keys.** Add a module-level
    `_KNOWN_FILE_HOOK_KEYS = frozenset({"name", "description", "command", "projects", "sidecars", "path_globs", "agent_name_globs", "ops", "timeout"})`
    and raise `ValueError` listing any key not in it. This matters: config layers are
    **not** validated against `sase.schema.json` at runtime (the schema is used by
    `src/sase/config/inventory.py` and tests only), so without this a stale `globs:`
    entry would parse to `path_globs=None`, which means _match every file_ — the
    `research-highlights` hook would silently start running `bob highlights create` on
    every file in the research sidecar. Skipping the hook with a warning is the correct
    failure mode.
  - Give the legacy key a targeted message so the fix is obvious, e.g.
    `"'globs' was renamed to 'path_globs'"` when `globs` is the offending key, and
    `"unknown field(s): <sorted, comma-joined>"` otherwise. The existing loader already
    catches `ValueError`, logs
    `Skipping invalid file hook '<name>' from config layer '<layer>': <message>`, and
    continues, so no loader change is needed.
- `_glob_matches` is already value-agnostic; keep it as-is and reuse it for agent names.
  Rename its `globs` parameter to `patterns` and `rel_path` to `value` so it does not
  read as path-only.
- `hook_matches_event`: after the existing project/sidecar/op checks, keep the
  normalized path check against `hook.path_globs`, and add
  `if hook.agent_name_globs and not _glob_matches(hook.agent_name_globs, event.agent_name or ""): return False`.
  Order the checks cheapest-first; both glob calls should only run when configured.

### 3. Engine — `src/sase/file_hooks/engine.py`

- Add `agent_name: str | None = None` as the final field of `CapturedFileEvent`, and
  pass it through `matching_event()`. Keep the default so plugin-side constructors of
  this exported dataclass do not break; correctness is enforced by the new tests
  instead.
- Add a fail-soft module helper:

  ```python
  def _current_agent_name() -> str | None:
      """Best-effort SASE agent attribution for events captured in-process."""
      try:
          from sase.agent.identity import resolve_local_agent_name

          return resolve_local_agent_name()
      except Exception:
          return None
  ```

  A lazy import inside the function matches how this module already imports
  `agent_workspace_dir` and `get_vcs_provider`.

- `_derive_commit_file_events`: add an `agent_name: str | None = None` keyword parameter
  and stamp it onto every `CapturedFileEvent` it builds. A default keeps the existing
  direct-call test working.
- `emit_commit_file_hook_events`: resolve `_current_agent_name()` once and pass it to
  `_derive_commit_file_events`. Resolving in the caller (not inside the derive helper)
  keeps the helper pure and directly testable.
- `capture_artifact_file_event`: stamp `agent_name=_current_agent_name()`.
- `emit_artifact_file_hook_event`: copy `captured_source.agent_name` onto the rebuilt
  event. This is the one place a forgotten field would silently drop attribution — the
  artifact path deliberately re-points `abs_path` at the stored artifact while keeping
  the source's matching context.
- `_batch_payload`: add `"agent_name": captured.agent_name` to each run dict.
- Leave `BATCH_SCHEMA_VERSION` at `1`. Bumping it would make `_load_batch` reject
  already-written, still-pending batches with "unsupported file-hook batch schema" and
  silently drop those hook runs. Adding an optional field is backward compatible as long
  as the runner reads it defensively (next step).

### 4. Runner — `src/sase/file_hooks/runner.py`

- In `_notify_run`, add an `f"agent: {run.get('agent_name') or '-'}"` line to `notes`
  (place it next to the existing `project:` / `repository:` lines). Use `.get()`, not
  `run["agent_name"]`, so a batch written by the previous version still executes and
  notifies.

### 5. CLI — `src/sase/main/file_hook_handler.py`

- `_filter_text`: replace the `("globs", hook.globs)` row with
  `("path_globs", hook.path_globs)` and `("agent_name_globs", hook.agent_name_globs)`,
  in that order, between `sidecars` and `ops`. Labels intentionally match the config
  keys exactly.
- Bump `FILE_HOOK_LIST_JSON_SCHEMA_VERSION` to `2`. `_json_payload` uses `asdict(hook)`,
  so the `-j/--json` output loses `globs` and gains two keys — a breaking shape change
  for any consumer.

### 6. Schema — `src/sase/config/sase.schema.json`

In the `fileHook` definition (around line 288):

- Rename the `globs` property to `path_globs`; keep the array-of-strings type and update
  the description to "Optional repo-relative path globs, including `!`-prefixed
  exclusions."
- Add `agent_name_globs`: array of strings, described as "Optional SASE agent-name globs
  matched against the agent that produced the event, including `!`-prefixed exclusions."
- `additionalProperties: false` already rejects the legacy `globs` key once renamed,
  which is what the new schema test asserts.

### 7. Docs — `docs/configuration.md` (`### file_hooks`, ~line 2040)

- Update the YAML example to the new field names, mirroring the real chezmoi hook:

  ```yaml
  path_globs: ["20*/**/*.md", "!20*/*/*__*.md"]
  agent_name_globs: ["!research.*.cld", "!research.*.cdx"]
  ```

- Field table: rename the `globs` row to `path_globs`, and add an `agent_name_globs` row
  (`list[string]`, not required, default "all agents", "SASE agent-name globs matched
  against the agent that produced the event; `!` prefixes a veto exclusion").
- Rename the "Glob matching" bullet to "Path glob matching" and add an "Agent-name
  matching" bullet covering: patterns match the resolved SASE agent name; positives are
  OR-ed and `!` vetoes; agent names contain no `/`, so `*` spans the whole name; **an
  event with no resolvable agent name matches a negative-only list but never a list
  containing a positive pattern**.
- Add an "Attribution" bullet: the agent name is resolved in the producing process from
  `$SASE_ARTIFACTS_DIR/agent_meta.json`'s `name`, falling back to `$SASE_AGENT_NAME`, so
  a commit made outside a SASE agent has no agent name.
- Add a short note that unknown `file_hooks` keys are rejected (hook skipped with a
  warning) and that `globs` was renamed to `path_globs`.

### 8. Tests

`tests/test_file_hooks.py`

- Update `_hook()` and `_event()` helpers for the renamed/added fields (`_event` gains
  an `agent: str | None = None` keyword).
- Rename the existing glob tests to say `path_globs` and update their kwargs.
- New: positive-only agent globs match only listed agents; negative-only agent globs are
  "everything except"; a mixed list ORs positives then applies the veto.
- New: `agent_name=None` matches a negative-only list and does **not** match a list with
  any positive pattern.
- New: path and agent filters are AND-ed (a matching path with an excluded agent does
  not match).
- New: a config entry using the legacy `globs` key is skipped, and the warning text
  mentions `path_globs`.
- New: a config entry with an arbitrary unknown key is skipped and warned about.
- New: an entry using `path_globs`/`agent_name_globs` parses into the right tuples.

`tests/test_file_hook_engine.py`

- Update `_hook()` and `_event()` helpers for the new fields.
- New: `emit_commit_file_hook_events` against a real temp repo with `SASE_AGENT_NAME`
  set writes `agent_name` into every run entry of the batch JSON.
- New: `agent_meta.json`'s `name` wins over `SASE_AGENT_NAME` when `SASE_ARTIFACTS_DIR`
  points at a directory containing it.
- New: with neither env var set, `agent_name` is `null` and a hook with negative-only
  `agent_name_globs` still runs.
- New: `capture_artifact_file_event` stamps the agent name, and
  `emit_artifact_file_hook_event` preserves it into the emitted batch (this is the
  regression test for the copy in step 3).
- New: a hook whose `agent_name_globs` excludes the current agent produces **no** batch
  file (`emit_file_hook_events` returns `None`).
- Use `monkeypatch.setenv`/`delenv` so the environment does not leak between tests.

`tests/test_file_hook_cli.py`

- Update `_hook()` to the new fields; assert the rendered filter block contains both
  `path_globs:` and `agent_name_globs:` rows, and that the JSON payload reports
  `schema_version == 2` with `path_globs`/`agent_name_globs` keys and no `globs` key.

`tests/test_config_schema.py`

- Update `test_config_schema_accepts_file_hooks` to the new field names plus an
  `agent_name_globs` entry.
- Add a rejection case asserting the legacy `{"globs": [...]}` shape now fails
  `Draft7Validator` (there is an existing parametrized invalid-hook test nearby to
  extend).

`tests/test_commit_workflow_hooks.py` and `tests/conftest.py`

- Check for `globs=`/`FileHookConfig(` construction and update. `conftest.py` only
  resets the loader cache, so it likely needs no change; confirm rather than assume.

Also grep the whole repo for remaining `globs=` / `"globs"` occurrences tied to file
hooks before finishing — `mentor_profiles`' unrelated `file_globs` must be left alone.

### 9. Verification

```bash
just install          # workspaces are ephemeral; required before anything else
just check
```

Run `just check-full` before landing: this touches config parsing plus two commit seams,
which is wider than the scoped test lane's usual blast radius.

Also sanity-check the rendered inventory by hand:

```bash
sase file-hook list
sase file-hook list -j
```

### 10. Update the chezmoi config — do this **last**

Open the repo through the skill, never by path:

```bash
sase repo open chezmoi -r "Update the research-highlights file hook to path_globs + agent_name_globs"
```

Edit `home/dot_config/sase/sase.yml` (the source for `~/.config/sase/sase.yml`) so the
hook reads:

```yaml
file_hooks:
  - name: research-highlights
    description:
      Render new research reports into Highlights PDFs for the Obsidian reading queue.
    command: bob highlights create
    sidecars: [research]
    path_globs: ["20*/**/*.md", "!20*/*/*__*.md"]
    agent_name_globs: ["!research.*.cld", "!research.*.cdx"]
    ops: [ADD]
    timeout: 120s
```

The `description` drops "consolidated" because the hook now intentionally covers
ordinary top-level research reports too.

Commit the chezmoi change with the `/sase_git_commit` skill, then `chezmoi apply` so
`~/.config/sase/sase.yml` picks it up.

**Ordering is not cosmetic.** `~/.config/sase/sase.yml` is global and shared by every
workspace, while the sase code change ships per-checkout:

- Old config + new code → the hook is skipped with a warning (safe, and this is exactly
  what step 2's unknown-key rejection buys).
- New config + old code → old code ignores both unknown keys, leaving `globs=None`,
  which means _match every file_. `bob highlights create` would then run against every
  markdown file in every research-sidecar commit.

So land and install the sase change first, and only then apply the chezmoi config.

## Out of scope

- Exposing the agent name to the hook command itself (e.g. a `SASE_FILE_HOOK_AGENT_NAME`
  environment variable). Only the final file path is passed today; keep it that way.
- Any back-compat alias that keeps accepting `globs`. The rename is intentional and the
  only real config is updated in the same change.
- Moving file-hook matching into `sase-core`. The subsystem is Python-only today; this
  change adds no new boundary crossing.

## Notes for the implementer

- The requested `agent_name_globs` patterns are anchored on the `.cld`/`.cdx` suffix, so
  a _retried_ researcher — `allocate_retry_name` in `src/sase/agent/names/_retry.py`
  renders `<base>.r@`, i.e. `research.7.cld.r1` — would **not** be excluded. Implement
  the globs exactly as specified; the user has been told about this and can widen them
  to `!research.*.cld*` / `!research.*.cdx*` later if a retry ever leaks a report
  through.
- `sase file-hook list` is the fastest way to confirm the loader parsed a config change
  — it prints the contributing layer per hook, so a silently skipped hook shows up as a
  missing row.
