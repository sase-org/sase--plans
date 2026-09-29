---
tier: epic
title: Private attachment sidecar and shared store
goal: Bead note attachments of at most the git tier are stored in a private attachments-private
  sidecar, uploaded before the bead store is published, and fetched on demand with
  honest availability badges. A missing store or an explicit local-only choice keeps
  the bytes on this machine and says so.
phases:
- id: sidecar_role
  title: Reserve the hidden private attachments-private sidecar
  depends_on: []
  size: medium
  description: 'sidecar_role: reserve the attachments-private sidecar role (repo <project>--attachments-private)
    as a hidden bare partial clone with default-private visibility, a default-no repo-init
    consent prompt, and a sase-side preflight test. Do not create GitHub repos or
    reserve the plain attachments name.'
- id: git_store
  title: Git blob store written with plumbing
  depends_on: []
  size: medium
  description: 'git_store: add GitAttachmentStore, a BlobStore over a bare partial
    clone that puts, checks, and fetches content-addressed objects with git plumbing,
    a writer lock, bounded fetch, and digest verification.'
- id: upload
  title: Placement, pre-publication upload, and outbox
  depends_on:
  - sidecar_role
  - git_store
  size: medium
  description: 'upload: place attachments with core policy and -L/--local-only, upload
    after the bead commit and before bead publication, and persist a durable outbox
    that attachment push and bead sync can drain.'
- id: fetch
  title: Lazy fetch, availability badges, and doctor
  depends_on:
  - upload
  size: medium
  description: 'fetch: lazily fetch under the auto-fetch cap for read, show, and path,
    render every availability badge, and add a sase doctor check for store reachability
    and outbox backlog.'
proposed_by: bbugyi200.athena.sase-1ck.5
parent_bead: sase-1ck.5
create_time: 2026-09-29 17:24:30
status: wip
bead_id: sase-1ck.5.1
---

- **PROMPT:** [prompts/202609/private_attachment_store.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/private_attachment_store.md)
- **PARENT:** [202609/bead_note_attachments.md](https://github.com/sase-org/sase--plans/blob/main/202609/bead_note_attachments.md)
- **BEAD:** [sase-1ck.5.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1ck/sase-1ck.5.1.md)

# Plan: Private attachment sidecar and shared store

This is the implementation plan for phase bead `sase-1ck.5` (`shared_store`) of epic
`sase-1ck`. The parent design is `plan:202609/bead_note_attachments.md`, Phase 5, plus
the 2026-09-29 amendment already recorded on the phase bead.
`research:202609/bead_attachment_audience/bead_attachment_audience.md` is not in this
workspace; the amendment text on the bead and in that Phase 5 block is the source of the
role rename.

Approving this plan creates a child epic under `sase-1ck.5`. Phase workers close only
their own phase beads. The child epic's land agent closes the child epic; the existing
upward cascade closes `sase-1ck.5`. No phase worker closes `sase-1ck`, `sase-1ck.5`, or
any other ancestor.

## Shared rules

- The reserved role key is `attachments-private`. The repo slug is
  `<project>--attachments-private`. Expose the constant
  `ATTACHMENTS_PRIVATE_SIDECAR_ROLE` next to `AGENTS_SIDECAR_ROLE` in
  `src/sase/sdd/_store_types.py` and re-export it from `sase.sdd.store`.
- Do not reserve the plain `attachments` role. It stays free for a future public store.
- Do not modify `sase-github`. Private-repo creation and visibility reporting already
  landed there. This work only adds the sase-side preflight test.
- No phase creates a real GitHub repository. Tests use a local bare remote.
- The store stays private-only. Do not add a `visibility` field to attachment
  descriptors, and do not add a public default for this role.
- Descriptors stay location-free. `origin` remains the machine name already recorded by
  `author_note_attachments` via `get_machine_name()`.
- Placement is the existing core function
  `sase_core_rs.attachment_placement(size_bytes, tiers, local_only)`. Tiers are
  `{"name": "git", "max_bytes": <git_max_bytes>}` dicts. Auto-fetch uses
  `sase_core_rs.attachment_should_auto_fetch(size_bytes, cap_bytes)`. Do not reimplement
  either policy in Python.
- Python owns I/O. Core already owns grammar, wire, and policy.
- Authoring stays behind the `bead_note_attachments` beta flag. Fetching, badges,
  `attachment list`, and `attachment path` are not flag-gated.
- Beads with no attachments do no attachment work: no git import, no network, no store
  import on that path. Pillow and git imports stay lazy.
- A preview or fetch failure never fails `show` or `read`. Prose still renders.
- Keep new modules under the `toobig` limit by splitting along the files named below.
  Run `just fix` before `sase tool run check`. Do not run raw `just check`.
- Do not create beads. Record discovered follow-up work as `PROPOSED FOLLOW-UP:` via
  `sase bead note` on the phase bead being closed.
- Out of scope for every phase: the rclone large tier, background uploads, progress
  bars, image previews, pager open, TUI, purge, cache prune, bead-page rendering, and
  removing the beta flag. Those are later phases of `sase-1ck`.

Remote object paths reuse `sase_core_rs.artifact_object_relpath`, which is
`files/objects/sha256/<xx>/<sha>`. Tombstones, when present, live at
`files/tombstones/sha256/<xx>/<sha>.json`. The local CAS layout in
`src/sase/bead/attachments/store.py` is unchanged.

## Phase 1: `sidecar_role`

Reserve the role and make its clone a hidden bare partial clone. This phase does not
upload or fetch attachment bytes.

### Role and schema

- Add `attachments-private` to `RESERVED_SIDECAR_ROLES` and to `HIDDEN_SIDECAR_ROLES` in
  `src/sase/_linked_repo_config.py`.
- Add it to the `repos.sidecar.builtin` `propertyNames` enum in
  `src/sase/config/sase.schema.json`, and to the `custom` `not` enum, so it cannot be
  declared as a document sidecar. Update the nearby descriptions that list only `plans`,
  `beads`, and `agents`.
- `document_sidecar_roles` excludes it, the same way it excludes `beads` and `agents`.
  It is not a document corpus and gets no entry in `BUILTIN_SIDECAR_REF_KIND`.
- Effective visibility is `private`. In `_merged_sidecar_entries_cached`, after the
  public `setdefault`, force this role to `private`. An explicit `visibility: public`
  for this role is a config error: preflight fails closed, and it is not rewritten into
  a public create. Do not change the public default for any other role, and do not
  change `SidecarInitSpec.visibility`'s dataclass default.
- Do not add this role to the plans/beads rows of `inject_default_linked_repos`. Do
  inject it for managed projects the way `agents` is injected, with `auto_clone: false`,
  `auto_sync: false`, and `visibility: private`, unless the user disabled it.
  `default_linked_repos: false` still suppresses it. Update
  `test_managed_project_injects_hidden_agents_sidecar_config` and every assertion that
  the hidden set is exactly `{"agents"}`.

### Hidden bare clone

`resolve_sidecar_clone_root` (`src/sase/sdd/_sidecar_init.py`) currently sends only
`agents` through `hidden_sidecar_clone_dir`. Send `attachments-private` through that
same helper, so the clone is
`hidden_sidecar_clone_dir(project_key, "attachments-private")` and never
`sase/repos/attachments-private`.

The clone is bare and partial:

```text
git clone --bare --filter=blob:none <remote> <hidden dir>
```

Do not use the working-tree path in `ensure_sidecar_sdd_clone` for this role. That
helper treats `(clone / ".git").is_dir()` as "already cloned", which is false for a bare
repo.

- `run_materialized_sidecars` must treat a bare repo (`HEAD` file plus an `objects`
  directory, and no `.git` directory) as materialized.
- Skip `_seed_sidecars` document seeding, README generation, and the artifact-link
  gitignore for this role. `repo init` prints the bare clone path, not `README.md`.
- When the clone has an unborn HEAD, create one empty root commit with plumbing
  (`commit-tree` of an empty tree, no parent) and push it, so later puts have a branch
  to update. The message is `chore(attachments): initialize private store` and contains
  no filename.
- A declined or missing role leaves no clone. Attachment commands then behave as "no
  shared store" (phase `upload`).

Audit every `HIDDEN_SIDECAR_ROLES` consumer. The new role must follow the agents
hidden-sidecar behavior except where this plan names a difference:

- Not cloned into workspaces and not exposed to launched agents
  (`axe/run_agent_runner_setup_linked_repos.py`, `_repo_inventory_workspaces.py`).
- Shown by `sase repo list` via the hidden-clone branch in `src/sase/repo_inventory.py`.
- Skipped by sidecar auto-sync (`src/sase/_sidecar_auto_sync.py`). The attachment store
  does its own fetch and push.
- Excluded from commit-finalizer dirty scans the same way agents is.
- `tools/ci_bootstrap_sidecars` has its own `HIDDEN_SIDECAR_ROLES` copy. Update that
  copy too, or it will try to materialize this role inside CI workspaces.

Do not copy agents-only publication text, prompt-archive seeding, or the agents memory
map onto this role.

### Repo init consent

Mirror `_confirm_agents_sidecar_creation`: default no, decline continues init without
this sidecar, non-interactive and EOF warn and continue. The prompt names the visibility
(`PRIVATE`) and says the repository will hold private bead attachment bytes for the
project. It is not the agents publication warning.

Onboarding's non-interactive path discards a missing `attachments-private` role the way
it discards a missing agents role, instead of treating it as a required document
sidecar.

### Preflight test

Add a test next to `tests/sdd_store/test_sidecar_init_preflight.py`. A fake
`preflight_sdd_sidecar` that reports `private` for an `attachments-private` spec with
`visibility: private` passes, and the captured options contain `sdd_visibility: private`
and `sdd_sidecar_suffix: attachments-private`. A fake that reports `public` raises the
existing "config requires private" error. Do not duplicate the generic artifacts-role
tests; this test pins the reserved role.

Also extend `test_sidecar_clone_root_keeps_agents_machine_scoped` so
`attachments-private` resolves under `SASE_HOME/projects/<key>/repos/` and `plans` still
resolves under the workspace.

### Done when

- Schema accepts `repos.sidecar.builtin.attachments-private` and rejects it under
  `custom`.
- The role is hidden, private, and absent from workspace checkouts.
- `sase repo list` can show the hidden clone.
- Declining the prompt leaves init successful and creates nothing.
- The preflight pair above passes.
- `sase tool run check` passes.

## Phase 2: `git_store`

Add `GitAttachmentStore` in `src/sase/bead/attachments/git_store.py`. It implements the
`BlobStore` protocol in `blob_store.py`. The constructor takes the bare repo path plus
the `describe()` label. It does not look up projects. Discovery of the hidden clone is
phase `upload`'s job, using the constant from phase `sidecar_role`.

`name` is `git`. `describe()` is the caller-supplied label, for example
`sase-org/sase--attachments-private (private)`.

### has

Resolve the tip from the cached remote-tracking ref. At most one bounded `git fetch` per
process refreshes it, with a timeout on the order of the existing SDD git command
timeout. Further `has` calls in that process do not fetch. A failed fetch keeps the last
cached tip and returns false for an unknown path; it does not raise. Presence is
`git rev-parse <tip>:<relpath>` against the tree, which a `blob:none` clone can answer
without downloading the blob.

### put

Hold an exclusive flock inside the bare git dir for the whole update.

1. `git hash-object -w` the local CAS file. The bytes are already digest-named; refuse
   to store a file whose hash is not `sha256`.
2. Load the tip into a temporary index with `GIT_INDEX_FILE`: `read-tree`,
   `update-index --add --cacheinfo`, `write-tree`.
3. `commit-tree` the new tree with parent tip. An unborn HEAD has no parent. The message
   is `chore(attachments): store <full sha256>` and contains no filename. If the path
   already points at the same blob, skip the commit and succeed.
4. Update the branch and `git push`. On non-fast-forward, fetch and rebuild from the new
   tip. Bound the retries (three is enough). Identical content-addressed paths do not
   conflict.

Writer retries may fetch again. The one-fetch cap applies to `has`, not to this conflict
retry.

### get

`git rev-parse <tip>:<relpath>`, then stream `git cat-file blob` into a CAS temp file.
That `cat-file` is the promisor fetch. Hash the stream. On mismatch do not install the
object; raise `BlobStoreError` with `transient=False`. On success, install through the
existing `LocalAttachmentStore` replace rules (same filesystem, flock, mode 0444).

### delete

Remove the path from a new tree with the same plumbing when it exists, and do nothing
when it does not. Do not write a tombstone. Purge remains a later phase; this method
only satisfies the protocol.

### Quarantine

Reuse the prompt-archive rule that a canonical object path must match its digest
(`src/sase/agents_sync/prompt_archive/archive_objects.py`, `is_valid_archive_object`).
Parameterize that check, or call a shared helper, instead of copying it. Do not copy the
worktree `git add` publisher in `git_ops.py`. A bare repo has no worktree to quarantine;
a digest mismatch on `get` is an error, not a relocated worktree file.

### Tests

Use a local bare remote with `uploadpack.allowFilter=true`, and two bare partial clones
(`--bare --filter=blob:none`). No network and no GitHub.

- Round-trip a small binary: `put` from one clone, `get` into a fresh CAS from the
  other, digest matches, view path keeps the filename.
- `has` is true after one fetch and does not fetch again in-process.
- A second put of the same digest is a no-op success.
- Concurrent puts of two different digests from the two clones both become reachable.
  The loser of the push retries and converges.
- A blob whose bytes do not match the path digest fails `get` and is not installed.
- Commit messages contain the digest and no filename.

### Done when

Those tests pass and `sase tool run check` passes. This phase has no CLI surface and no
config keys.

## Phase 3: `upload`

Wire placement and upload into the authoring commands. Config keys added here, and only
these, go in `src/sase/default_config.yml`, `sase.schema.json` under `bead.attachments`,
`src/sase/bead/config.py`, and `docs/configuration.md`:

- `git_max_bytes`: integer, default `52428800` (50 MiB). A missing or malformed value,
  or a value above `99614720` (95 MiB), fails open to the default, matching the other
  bead accessors.
- `require_upload`: boolean, default false. A missing or malformed value fails open to
  false.

Do not add `auto_fetch_max_bytes`, `large_store`, or `background_upload_min_bytes` in
this phase.

### When there is a shared store

A shared store exists when the `attachments-private` hidden bare clone exists and has a
remote. Build one `GitAttachmentStore` pointed at that clone. The describe label is
`<owner>/<repo> (private)` from the recorded sidecar identity, falling back to
`<project>--attachments-private (private)`.

No clone means there is no shared store. Placement does not call core with an empty tier
list. The attachment is local by design, nothing is queued, and every write echo says
so. `require_upload: true` with no store aborts before the bead event is written.

### Placement

Call `attachment_placement` with the single git tier and the command's `local_only`
flag.

- `local_only` (the user passed `-L`) returns `local_only`. Do not upload and do not
  enqueue.
- A size the git tier accepts returns store `git`. Upload that object.
- No tier accepts the size: fail before any bead write unless `-L` was passed. On a TTY,
  ask y/N to keep it local-only; yes continues as `-L`, no or EOF writes nothing. A
  non-TTY fails with the size and the `-L` hint.

`-L/--local-only` is free on `note`, `close`, `update`, `+1`, and `attach` today
(`update`'s `-d` is `--description`, so do not put download there). Recheck those
parsers before adding the flag. `bead tree` already uses `-L` for `--levels`; that is a
different parser and stays as it is.

Thread the flag through the note, close, update, +1, and attach handlers, and through
the TUI add-note save call of the authoring service. The flag does nothing when
`bead_note_attachments` is off.

### Upload timing

`bead_store_mutation` in `src/sase/bead/cli_common.py` commits inside the store lock,
then calls `_push_committed_bead_store` after the lock is released. Register pending
uploads on the mutation. After a successful commit and outside the lock, run them before
`_push_committed_bead_store`.

Default order: ingest (already done by authoring), append and commit the bead event,
upload, then publish the bead store. A failed upload appends the digest to the outbox
and still lets the bead publication proceed. The echo shows `pending upload` for that
object.

`require_upload: true` uploads before the append. Any `put` failure aborts with nothing
written to the bead store and nothing left queued. A successful `put` followed by a
failed append may leave the content-addressed blob in the sidecar; do not roll that blob
back in this phase.

The write echo gains the destination label from `describe()`, the word `private`, and
the elapsed upload time, beside the existing descriptor row. Local-only and no-store
rows say they stayed on this machine instead of quoting a remote. Agents get the same
facts as plain text. Reuse `attachment_presentation` glyphs rather than a second
formatter.

### Outbox

Path: `<sase home>/projects/<key>/attachment-upload-outbox.json`, via
`sase_projects_dir`, so `SASE_HOME` is honored. Flock plus atomic replace, following
`src/sase/agents_sync/publication_outbox_store.py`. Records are digest, size, store name
(`git`), and origin machine. No filenames.

Drain it:

- Opportunistically, with a short time bound, at the start of an attachment-writing
  command, before the new upload.
- At the start of `_run_locked_sync` in `src/sase/bead/_sync_worker_run.py`. A drain
  failure is logged and never fails bead publication.
- From `sase bead attachment push [<id>]`. With an id, drain only digests referenced by
  that bead. Without an id, drain the project outbox.

`push` also promotes local-only objects once a store exists and `attachment_placement`
accepts their size: objects in the local CAS that are referenced by a current note,
absent from the outbox, and absent from the git store. A size the store still rejects
stays local-only.

Place the command alphabetically under `sase bead attachment` and add it to the
attachment help examples. It is available with the flag off so a queued upload can drain
after the flag is disabled; it does not author notes.

### Tests

Both flag states:

- Flag off: a note containing `@./file` keeps today's text behavior, writes no outbox
  entry, and does not invoke git.
- Flag on, two temp homes sharing one local bare remote: a failed `put` queues the
  digest, the note exists, and a later `attachment push` drains it so the other home can
  see the object. Use a fake or a broken remote for the failure, then repair it.
- A file larger than `git_max_bytes` fails with nothing written when `-L` is absent and
  stdin is not a TTY. `-L` writes the note and does not enqueue.
- `require_upload: true` and a failing `put` leave the bead store unchanged and the
  outbox empty.
- No hidden clone: the echo says the attachment stayed local, and the outbox stays
  empty.

Do not require the full second-home `read` here. Phase `fetch` owns that.

### Done when

Those tests pass and `sase tool run check` passes.

## Phase 4: `fetch`

Add `auto_fetch_max_bytes` the same way phase `upload` added its keys: default
`26214400` (25 MiB), fail open to that default when missing or malformed. Document it.
Do not add the large-store or background-upload keys.

### Fetch

For `read`, `show`, and `attachment path`, resolve each attachment in this order: local
tombstone, local CAS hit, then the git store.

- `attachment_should_auto_fetch` is true: `GitAttachmentStore.get` into the CAS, then
  materialize the extension-preserving view. `path` prints that absolute path.
- The cap refuses automatic fetch: do not download. `show` and `read` render the
  not-downloaded badge. `path` still fetches, because explicit path always fetches.
- `-d/--download` on `show` and `read` lifts the cap for that invocation. `-d` is free
  on both parsers today (`-c`, `-f`, `-N`, `-p`, `-P`, `-r` on read, `-s`, `-w`).
  Recheck before adding it.

`attachment list` does not fetch. It reports the state it can see.

A fetch error, including a digest mismatch, is caught at the presentation boundary. The
command still exits 0 for `show` and `read`.

### Availability

Replace `attachment_availability` in `src/sase/bead/attachment_presentation.py`. It
currently returns only `cached` or `unavailable`. Compute the state from the local CAS,
the local tombstone directory, the outbox, store `has`, the origin machine, and the cap.
Render these badges, and only these:

| State            | Badge                                                                           |
| ---------------- | ------------------------------------------------------------------------------- |
| `cached`         | none                                                                            |
| `remote`         | none while a fetch is about to run; after a skipped fetch this is not the label |
| `not_downloaded` | `⇣ not downloaded · <size> — sase bead attachment path <id> <name>`             |
| `pending_upload` | `⇡ pending upload (<origin>)`                                                   |
| `local_only`     | `⚠ only on <origin>`                                                            |
| `unavailable`    | `✕ unavailable offline`                                                         |
| `purged`         | `(purged)`                                                                      |
| `corrupt`        | `‼ digest mismatch`                                                             |

Derivation:

- Local tombstone file, or a remote tombstone object: `purged`. Do not fetch. Do not
  implement purge.
- Local object fails `LocalAttachmentStore.verify`: `corrupt`.
- `get` digest mismatch: `corrupt`, object not installed.
- Local object present and store `has` is true: `cached`.
- Digest is in the outbox: `pending_upload`, machine name from `origin`.
- Local object present, not in the outbox, store does not have it: `local_only`.
- Local object absent, store has it, size above the cap, and this command is not an
  explicit fetch: `not_downloaded`.
- Local object absent, store has it, and this command will fetch: fetch, then `cached`
  or `corrupt`.
- Otherwise: `unavailable`.

JSON `notes[].attachments[]` and the +1 evidence attachments gain `availability` and
`local_path` (only when cached). They already do this for the two old states in
`cli_detail_json.py` and `cli_attachment.py`. Extend those fields; do not add bytes.
Text detail in `cli_detail_sections.py` uses the same badges.

`remote` is the pre-fetch condition "store has it and we are allowed to fetch". Callers
that fetch never leave the user-visible state as `remote`.

### Doctor

Add `src/sase/doctor/checks_attachment_store.py` and register it from
`src/sase/doctor/runner.py` beside the other project checks.

- Id `project.attachment_store`. Pick an unused alias by searching existing `CheckSpec`
  aliases; do not reuse one.
- Skip with a clear summary when the current project has no `attachments-private` clone.
- Warn when the clone exists but `git rev-parse` of the tip fails or the one bounded
  fetch fails.
- Warn when the outbox has entries, and include the count. A malformed outbox file is a
  warning, not a crash.
- OK when the clone's tip resolves and the outbox is absent or empty.

This is `sase doctor`, not `sase bead doctor`. No `--fix-attachments` in this phase.

### Acceptance tests

Two temp `SASE_HOME` directories, one shared local bare remote, two bare partial clones.
Never the real project bead store or the real `~/.sase` attachment CAS.

- Home A attaches a small file. Home B `read`s the bead and gets a digest-verified,
  extension-preserving path. The badge is cached.
- An object larger than `auto_fetch_max_bytes` but within `git_max_bytes` shows
  `not_downloaded` on B until `read -d` or `attachment path`, which then verifies the
  digest.
- Concurrent attaches of different files on A and B both end up fetchable from either
  home.
- A remote blob altered after `put` so its bytes no longer match the digest is reported
  `corrupt` and is not installed into B's CAS.
- The doctor check is OK with an empty outbox, warns with a queued item, and skips with
  no clone.
- Flag off: `read` of a pre-existing attachment note still renders badges and can fetch.
  Flag off does not author new attachments.

### Done when

Those tests pass and `sase tool run check` passes. A bead with no attachments still
renders with no git subprocess and no attachment-store import on that path; cover that
with one test that spies on the git launcher or the store constructor.
