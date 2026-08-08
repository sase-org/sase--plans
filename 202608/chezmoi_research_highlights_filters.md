---
tier: tale
title:
  Migrate the chezmoi research-highlights file hook to path_globs + agent_name_globs
goal:
  The chezmoi-managed global `sase.yml` `research-highlights` file hook uses the new
  `path_globs` and `agent_name_globs` filter fields in place of its legacy `globs:`
  line, is committed to the chezmoi repo, and is applied to `~/.config/sase/sase.yml` so
  the hook loads again and fires on every research report except the two
  pre-consolidation `#research_swarm` researchers' throwaway reports.
proposed_by: bbugyi200.athena.vc.f1
create_time: 2026-08-07 21:51:57
status: done
---

- **PROMPT:**
  [prompts/202608/chezmoi_research_highlights_filters.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/chezmoi_research_highlights_filters.md)

# Plan: chezmoi `research-highlights` → `path_globs` + `agent_name_globs`

## Problem

This is step 10 of the already-implemented plan
`~/.sase/plans/202608/file_hook_agent_name_globs.md`. That plan deliberately deferred
the chezmoi config edit until the sase code change landed, because the ordering matters.
The code change has now landed, so the deferred half is safe to do — and is in fact
overdue.

The chezmoi source `home/dot_config/sase/sase.yml` still reads:

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

Because the landed code renamed `globs` to `path_globs` and now rejects unknown
`file_hooks` keys, **the hook is currently disabled**. Verified live from two different
installs:

```
$ /home/bryan/.local/bin/sase file-hook list          # global uv-tool install
Skipping invalid file hook 'research-highlights' from config layer 'user': 'globs' was renamed to 'path_globs'
No file hooks configured.

$ .venv/bin/sase file-hook list                       # this workspace's venv
Skipping invalid file hook 'research-highlights' from config layer 'user': 'globs' was renamed to 'path_globs'
No file hooks configured.
```

This is the intended fail-soft behavior (skip loudly rather than silently match every
file), but it means no research report is being rendered into a Highlights PDF right
now. Applying this plan restores the hook _and_ broadens it as originally designed.

## Goal

Replace the single `globs:` line with the two filter lines the user specified:

```yaml
path_globs: ["20*/**/*.md", "!20*/*/*__*.md"]
agent_name_globs: ["!research.*.cld", "!research.*.cdx"]
```

Then commit the chezmoi change and `chezmoi apply` it so `~/.config/sase/sase.yml` picks
it up.

## Preconditions — all verified, do not re-derive

Each of these was checked before this plan was written. The implementer should
spot-check the first one (it is the safety gate) and may take the rest as given.

1. **The sase code change is landed and installed.** Commit
   `6ef7d14d5 feat(file-hooks)!: add agent_name_globs and rename globs to path_globs` is
   an ancestor of both `HEAD` and `origin/master` (it sits 7 commits back from
   `050c9477c`).

2. **The global `sase` has the new code.** `~/.local/bin/sase` →
   `~/.local/share/uv/tools/sase/bin/sase`, whose `_editable_impl_sase.pth` points at
   `/home/bryan/projects/github/sase-org/sase/src`. The `sase file-hook list` output
   above proves that checkout already contains the rename. This is the install every
   non-`sase` project's agents use, so the dangerous "new config + old code" combination
   is closed for them.

3. **No running agent is on an old workspace.** All 21 live agents occupy `sase`
   workspaces 10–17 (or are `WAITING`/`QUEUED` with no workspace yet), and workspaces
   10–18 all contain `path_globs` in `src/sase/config/file_hooks.py`. Workspaces 19–25
   are still on pre-`6ef7d14d5` commits, but every one of them is idle with a clean
   working tree, and claiming a workspace resets it to `origin/master` ("Cleaning
   workspace… Updating workspace to origin/master") — so they get the new code before
   any agent runs there.

4. **`~/.config/sase/sase.yml` is byte-identical to the chezmoi source right now**, so
   `chezmoi apply` will be a clean single-file update with no unrelated drift and no
   merge prompt.

5. **Nothing else in the chezmoi repo references this hook.** The only two hits for
   `research-highlights` / `highlights create` are lines 9 and 11 of that same file.
   `home/dot_config/sase/sase_athena.yml` and `sase_kellys_mbp.yml` declare no
   `file_hooks`, so there is no overlay layer to keep in sync.

6. **No formatter will reflow the edit.** chezmoi's `just fmt` covers Python, Lua, and
   Markdown only; `.prettierignore` is irrelevant here; and the `file_hooks` block is
   not inside a `keep-sorted` region. Flow-style `["a", "b"]` lists match the file's
   existing style.

## Verified matching semantics

Re-measured **against the landed code** (not the pre-implementation estimate) by
constructing the exact proposed `FileHookConfig` and calling `hook_matches_event`:

| agent name            | repo-relative path     | matches | why                                     |
| --------------------- | ---------------------- | ------- | --------------------------------------- |
| `research.7.final`    | `202608/foo.md`        | `True`  | **newly matched** — the whole point     |
| `research.7.cld`      | `202608/foo.md`        | `False` | vetoed by `!research.*.cld`             |
| `research.7.cdx`      | `202608/foo.md`        | `False` | vetoed by `!research.*.cdx`             |
| _(none)_              | `202608/foo.md`        | `True`  | negative-only list admits unattributed  |
| `research.7.final`    | `202608/foo/foo.md`    | `True`  | the consolidated report; matched before |
| `research.7.final`    | `202608/foo/foo__a.md` | `False` | vetoed by `!20*/*/*__*.md`              |
| `bbugyi200.athena.v8` | `202608/foo__a.md`     | `True`  | known consequence, see below            |
| `research.7.image`    | `202608/a/b/c.md`      | `True`  | **newly matched** via `**`              |
| `research.7.cld.r1`   | `202608/foo.md`        | `True`  | retried researcher is **not** vetoed    |
| `research.7.final`    | `README.md`            | `False` | outside `20*/`                          |

Two rows are deliberate accepted behavior, already flagged to and accepted by the user
in the parent plan. **Do not "fix" either by editing the requested globs** — they are
the user's explicit spec:

- `202608/foo__a.md` matches, because the `!20*/*/*__*.md` veto requires exactly one
  intermediate directory. Those files only exist between a researcher writing one and
  `.final` moving it under `<name>/`, and both researchers are vetoed by agent name, so
  it is harmless.
- `research.7.cld.r1` (a retried researcher, per `allocate_retry_name` in
  `src/sase/agent/names/_retry.py`) is **not** vetoed, because the patterns anchor on
  the `.cld`/`.cdx` suffix. If a retry ever leaks a report through, the fix is to widen
  to `!research.*.cld*` / `!research.*.cdx*` later — not now.

## Implementation

### 1. Open the chezmoi repo through the skill

Never touch chezmoi by path. Use the `/sase_repo` skill:

```bash
sase repo open chezmoi -r "Update the research-highlights file hook to path_globs + agent_name_globs"
```

Use the path it prints as the only path for reads and writes. From a numbered `sase`
workspace it resolves to `<workspace>/sase/repos/linked/chezmoi`.

### 2. Edit `home/dot_config/sase/sase.yml`

Replace **only** line 13 (the `globs:` line) with the two new filter lines, keeping
surrounding lines and indentation untouched. The `file_hooks` block becomes:

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

Field order matters only for readability, but put `path_globs` then `agent_name_globs`
between `sidecars` and `ops` so the file mirrors both `docs/configuration.md` and the
`fileHook` property order in `src/sase/config/sase.schema.json`.

**One judgment call for the reviewer.** The `description` above drops the word
"consolidated", because the hook now intentionally covers ordinary top-level research
reports too — leaving it in would make the description actively wrong. This wording
change was specified in step 10 of the approved parent plan, but this request only asked
for the `globs:` line, so it is called out explicitly. If the reviewer prefers a
strictly minimal diff, keep the existing `description` verbatim and change nothing but
the filter lines; everything else in this plan is unaffected.

### 3. Commit the chezmoi change

Use the `/sase_git_commit` skill from the chezmoi checkout. Do not run raw `git commit`.

A `feat(file-hooks):` or `chore(config):` subject is appropriate; the body should note
that this is the config half of sase `6ef7d14d5` and that the hook was skipped-with-a-
warning until now.

### 4. Apply

```bash
chezmoi apply
```

Then confirm the live file actually changed:

```bash
grep -n "path_globs\|agent_name_globs\|globs" ~/.config/sase/sase.yml
```

Expect exactly the two new lines and **no** bare `globs:` line.

### 5. Verify

```bash
sase file-hook list
sase file-hook list -j
```

Success criteria, all of which must hold:

- The
  `Skipping invalid file hook 'research-highlights' … 'globs' was renamed to 'path_globs'`
  warning is **gone**.
- `research-highlights` appears as a row, attributed to config layer `user` (a silently
  skipped hook shows up as a missing row, which is why this check is the real gate).
- The rendered filter block shows both a `path_globs:` and an `agent_name_globs:` row
  with the expected values.
- `-j` reports `schema_version` 2, with `path_globs` and `agent_name_globs` keys and no
  `globs` key.

Run the same two commands with the workspace venv binary
(`.venv/bin/sase file-hook list`) as well, to confirm the config parses identically
under a workspace install and not just the global one.

No `just check` run is required: this plan changes no file in the sase repo. If the
implementer ends up touching anything under the sase workspace checkout, that exemption
lapses and `just check` applies as usual.

## Rollback

Single-file, fully reversible. `git revert` the chezmoi commit (or restore the `globs:`
line) and re-run `chezmoi apply`. The worst case while reverted is the current state:
the hook is skipped with a loud warning and no Highlights PDFs are generated — no data
loss, no destructive command runs.

## Risks

- **Low: an idle old workspace is claimed mid-flight.** Workspaces 19–25 predate
  `6ef7d14d5`. If one were somehow used _without_ being reset to `origin/master`, its
  old code would ignore both new keys, leaving `globs=None`, which means _match every
  file_ — `bob highlights create` would run on every markdown file in every
  research-sidecar commit. Claiming resets the workspace to `origin/master`, and all
  seven are idle with clean trees, so this is closed in practice. Worth one glance at
  `sase agent list` right before applying; if any agent is running in workspace 19–25,
  wait for it to finish.
- **None from the sase side.** This plan does not modify the sase repo, so it cannot
  regress the code that just landed.

## Out of scope

- Widening the agent globs to cover retried researchers (`!research.*.cld*`).
  Deliberately deferred; see the semantics section.
- Any change to `src/`, `docs/`, or the JSON schema in the sase repo — all of that
  landed in `6ef7d14d5`.
- Adding `projects: [sase]` to the hook. The documented example in
  `docs/configuration.md` includes it, but the real hook intentionally does not restrict
  by project, and narrowing it is not part of this request.
