---
tier: tale
title: Lazy attachment-store discovery on show and read
goal: Show and read of a bead with no attachments do no attachment-store work, then
  epic sase-1ck.5.1 and its parent phase sase-1ck.5 are closed.
size: small
proposed_by: bbugyi200.athena.sase-1ck.5.1.land
bead: sase-1ck.5.1
status: done
---

- **PARENT:**
  [202609/private_attachment_store.md](https://github.com/sase-org/sase--plans/blob/main/202609/private_attachment_store.md)
- **BEAD:**
  [sase-1ck.5.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1ck/sase-1ck.5.1.md)

# Plan: Lazy attachment-store discovery on show and read

Epic `sase-1ck.5.1` is otherwise complete. Its four phases are closed, their commits are
on master (`aa61902a9e`, `e300173faf`, `c8796af46d`, `978f6ebdb6`), and
`sase bead epic-symbols sase-1ck.5.1` lists nothing. This tale finishes the one
done-when gap and then closes the epic. Do not reopen the closed phases.

## Gap

`sase bead show` and `sase bead read` always enter `fetch_context()` in
`src/sase/bead/cli_query.py`. That calls `build_fetch_context()` in
`src/sase/bead/attachments/fetch.py`, which imports `sase.bead.attachments.upload` and
calls `discover_shared_store()`. When the current project has an `attachments-private`
hidden clone with a remote, that discovery runs `git config --get remote.origin.url` and
constructs `GitAttachmentStore` before the command knows whether the bead has
attachments.

The epic plan's shared rule is that a bead with no attachments does no attachment work:
no git import, no network, and no store import on that path. Phase `fetch`'s done-when
requires one test that spies on the git launcher or the store constructor. JSON
enrichment already returns early when `issue.notes` have no attachments
(`cli_detail_json.py`), and the text renderer only imports attachment presentation
inside `if note.attachments`. The outer fetch context does not.

`attachment list` and `attachment path` must keep discovering the store, because they
report availability or always fetch. `attachment open` stays on
`fetch_context(mode="never")`. Do not make open fetch. Pager open, image previews, the
TUI, and the rclone tier stay out of scope.

## Implementation

Keep `FetchContext` mutable. Add `discover: bool = False`. A context built by tests or
by the `context is None` fallback in `attachment_state` stays `discover=False` and must
not replace an injected `store`.

`fetch_context()` installs a context with the requested mode, the current auto-fetch
cap, and `discover=True`. It does not import `upload` and does not call
`discover_shared_store` or `build_fetch_context`.

Add `_ensure_discovered(context)`. If `discover` is false, return. Otherwise set
`discover` false first, then run today's discovery body (project key, shared store,
outbox) and copy `project_key`, `store`, and `outbox` onto `context`. Leave `mode`,
`cap_bytes`, `failed`, and `corrupt` alone. A discovery failure still clears the flag
and leaves `store` as `None`, matching today's never-raise behavior.

Call `_ensure_discovered` at the start of `attachment_state` and `resolve_badge_origin`,
after the ambient context is resolved. Direct `FetchContext(store=...)` callers,
including the state-machine tests, never discover.

`build_fetch_context` may remain the eager helper used only by `_ensure_discovered`.
Nothing else should call it on the show/read path.

## Test

Add one test in `tests/test_bead/test_attachment_fetch.py` using the existing
`project_dir`, `work_dir`, `remote`, `_plant_hidden_clone`, `_create_plan`, and
`_read_full` helpers.

Plant the hidden clone into the test home before the read, so today's eager path would
construct `GitAttachmentStore`. Create a plan bead and do not attach a file. Monkeypatch
`sase.bead.attachments.upload.clone_has_remote` and
`sase.bead.attachments.git_store.GitAttachmentStore` to record calls. Read that bead
with `_read_full`. The command exits 0 and neither spy is called.

Do not assert that `git` is absent from the process: the bead store itself uses git. Spy
only the attachment-store discovery entry points above.

## Verify

Run:

```bash
just fix
.venv/bin/python -m pytest tests/test_bead/test_attachment_fetch.py tests/test_bead/test_attachment_upload.py tests/test_bead/test_git_attachment_store.py -q
```

`just _lint-patch-stitch-terminology` is already red on a clean tree (14 unclassified
hits in the sase-core `at_bearing_notes.jsonl` fixture) and is ready task `sase-1cv`. Do
not fix it here. Do not run `just check-full`.

## Closeout

Do this in the same turn as the code. Do not wait for this turn's commit SHA, push, or
CI.

1. Run `sase bead epic-symbols sase-1ck.5.1`. There should be no entries. This change
   adds no public symbol that needs a whitelist entry. If an entry is listed, resolve it
   (wire it, privatize it, or delete it). Do not re-key it and do not add a new
   `--epic-symbol` line.
2. Close the epic:

```bash
sase bead close sase-1ck.5.1 --note "<what you verified>"
```

The note must say that the four child phases were already closed and their commits
implement the sidecar, git store, upload/outbox, and fetch/badges/ doctor work; that
show/read of a bead with no attachments no longer discovers the store, covered by the
new spy test; that the focused attachment tests passed; that the terminology follow-ups
were +1'd onto `sase-1cv` and the symvision private-import follow-up stays the existing
`sase-1ck` discovered issue from `sase-1ck.7`; and that concurrent commits since
`aa61902a9e` did not need further integration (the toobig `git_store` package split is
what fetch already imports, revision-pin left the private visibility force in place, and
show previews run after status lines that fetch). If close is rejected for leftover
`--epic-symbol` entries, fix those and close again. Do not use `--force`. 3. Run
`just symvision`. Imports of `_kitty_graphics_support` and `_roster_for_issue` from
`sase-1ck.7` may still fail it. Those are not this tale's symbols. If the only findings
are those two, record that in the close note if it is not already there, and do not add
`--epic-symbol` lines. If symvision reports a symbol this tale added, fix it and
re-run. 4. Set `status: done` in the frontmatter of
`sase/repos/plans/202609/private_attachment_store.md`. 5. The parent of `sase-1ck.5.1`
is phase bead `sase-1ck.5` (title: Private attachments sidecar, upload outbox, and lazy
fetch), whose parent is epic `sase-1ck`. Read `sase-1ck.5` and confirm this epic's work
matches that phase: hidden bare `attachments-private` sidecar, git blob store, placement
with `-L` / pre-publication upload / outbox / `attachment push`, capped lazy fetch,
availability badges, and the `project.attachment_store` doctor check. Then close only
that phase:

```bash
sase bead close sase-1ck.5 --note "<what you verified against the phase>"
```

Do not close `sase-1ck`. Do not use `--force`. Leave the containing epic to its land
agent.
