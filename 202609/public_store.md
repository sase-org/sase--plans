---
tier: tale
title: Public attachments sidecar, routing, and anonymous reads
goal:
  Public bead attachments land in a reserved hidden attachments sidecar that a
  zero-credential machine can fetch, private bytes stay on the private tiers, and a
  secret-scanning rejection is permanent and visible.
size: medium
proposed_by: bbugyi200.athena.sase-1d5.4
bead: sase-1d5.4
create_time: 2026-09-30 11:55:12
status: wip
---

- **PARENT:**
  [202609/public_bead_attachments.md](https://github.com/sase-org/sase--plans/blob/main/202609/public_bead_attachments.md)
- **BEAD:**
  [sase-1d5.4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1d5/sase-1d5.4.md)

<!-- sase:links:start -->

## Links

| Relation     | Artifact                                                                  | Why                                                                            |
| ------------ | ------------------------------------------------------------------------- | ------------------------------------------------------------------------------ |
| derives-from | [plan:202609/public_bead_attachments.md][1]                               | Phase public_store of the public-by-default bead attachments epic              |
| derives-from | [research:202609/bead_attachment_audience/bead_attachment_audience.md][2] | Store topology, anonymous reads, and the GH013 lifecycle this phase implements |

[1]: https://github.com/sase-org/sase--plans/blob/main/202609/public_bead_attachments.md
[2]:
  https://github.com/sase-org/sase--research/blob/main/202609/bead_attachment_audience/bead_attachment_audience.md

<!-- sase:links:end -->

# Plan: Public attachments sidecar, routing, and anonymous reads

Implement bead `sase-1d5.4` (phase `public_store` of epic `sase-1d5`) in this sase
checkout. The audience decision, wire `visibility`, provenance probe, and core layout
helpers already exist. This plan adds the public store and routes bytes by the audience
those helpers already chose.

`attachment_public_object_relpath`, `attachment_canonical_extension`, and
`attachment_object_digest_from_relpath` are on the pinned `sase_core_rs`. Do not edit
sase-core, sase-github, versions, or changelogs.

## Outcome

- A public object written on one home is read on a second home that has only the public
  remote.
- Private objects and objects over `public_max_bytes` never reach the public remote.
- A missing public store queues `⇡ pending upload` and drains once the store exists. It
  never falls back onto `git` or `large`.
- A miss in one store is not a digest mismatch. Lazy fetch discovery sees every store.
- `GH013` (and the two sibling push-rejection strings below) resets the local ref,
  blocks that outbox entry, keeps a private copy, and narrows note descriptors to
  private.
- `sase repo init` offers the public repo only while `public_bead_attachments` is
  enabled, only when beads are public, and only after an anonymous probe agrees. Reads,
  discovery, read-side clone creation, and outbox drains stay ungated.

## Out of scope

Leave these to their owners:

- `sase-1d5.5` publish, unpublish, doctor rescans, and the
  private-store-is-anonymously-readable finding (`sase-1db`).
- `sase-1d5.6` page embeds. This phase only records which URL form embeds.
- `sase-1d5.7` TUI and `sase-1d5.8` flag removal and the audience-model doc rewrite.
- Do not close `sase-1d5` or any ancestor. Do not create beads. Record a new defect as
  `sase bead note sase-1d5.4 'PROPOSED FOLLOW-UP: …'`.

`sase-1da` is in scope for the discovery rewrite: a per-role store whose merged sidecar
entry has `disabled: true` is absent even when the hidden clone still exists. Apply that
to the public role and to `attachments-private`, in discovery and in
`project.attachment_store`. Update the two docs sentences that currently say a disabled
private role keeps uploading through an existing clone (`docs/beads.md` storage section
and `docs/init.md` attachments-private paragraph). Leave `sase-1da` open.

## Step 1 — Page-embed spike, then a bead note

Read-only. Push nothing. Open the sidecars with `sase repo open`.

1. `curl -I` an extensionless object under `sase--agents/files/objects/…` and a `.png`
   in `sase--research`.
2. Render `![x](<blob URL>?raw=true)` and the raw URL through `gh api markdown`. Fetch
   the camo URL GitHub returns and record the content type.
3. Decide which URL form bead pages should embed. If neither embeds an image, the
   finding is that pages link instead of embedding.

Record the finding with `sase bead note sase-1d5.4`. The extension-preserving layout
below stays either way: raw content type depends on it. Phase `presentation` consumes
the note. Do not implement page HTML here.

## Step 2 — Reserved `attachments` role

Add `ATTACHMENTS_SIDECAR_ROLE = "attachments"` beside `ATTACHMENTS_PRIVATE_SIDECAR_ROLE`
in `src/sase/sdd/_store_types.py`. It is reserved and hidden, and it is not a document
role (`document_sidecar_roles` excludes it, same as beads, agents, and
`attachments-private`).

Thread the constant through:

- `src/sase/_linked_repo_config.py`: `HIDDEN_SIDECAR_ROLES`; builtin order `plans`,
  `beads`, `agents`, `attachments`, `attachments-private`; a default description
  ("Hidden sidecar that stores public bead attachment bytes for this project.");
  `inject_default_linked_repos`.
- `src/sase/config/sase.schema.json`: add `attachments` to the builtin key enum and to
  the custom-role exclusion enum, and update the surrounding descriptions that list the
  reserved roles. Same comment in `src/sase/default_config.yml`.
- `src/sase/repo_inventory.py`: hidden-clone description branch for the new role.
- `src/sase/main/_repo_init_sidecars.py`: `_OPTIONAL_HIDDEN_SIDECAR_ROLES`, and init
  echo prints the bare clone path (no README), same as the private role.
- `docs/configuration.md`: the reserved-role lists and the visibility row. One accurate
  sentence each. Leave the audience-model rewrite to `ga`.

Visibility is forced to `public` in `_merged_sidecar_entries_cached`, the mirror of the
private role's forced `private`.

Inject the role from `inject_default_linked_repos` only when the beads sidecar
visibility in that same config is `public` (default `public`; non-sidecar bead storage
counts as private — `bead_store_visibility()` in
`src/sase/bead/attachments/provenance.py` is the rule, applied to the config being
merged). The existing collision skip still drops the injection when the user already
configured an `attachments` role or slug. `disabled: true` on the builtin role
suppresses it the same way the other hidden roles work. Injection is not flag-gated.

## Step 3 — Creation on `sase repo init`

Flag-gated with `audience_enabled()` from `src/sase/bead/attachments/audience.py`. While
the flag is off, do not prompt and do not create the repo (no warning). While it is on:

- Consent copy states `Repository visibility: PUBLIC.` and says who can read it: anyone
  who can read the beads sidecar. The prompt defaults to yes (`[Y/n]`); an empty answer
  accepts. Declining continues init without the sidecar.
- Without a TTY, skip the role with a warning. `--yes` does not authorize creation (same
  rule as the other hidden sidecars).
- Before creating, probe the beads remote with `resolve_remote_visibility`. Unless the
  probe returns `public`, refuse and name
  `repos.sidecar.builtin.beads.visibility: private`. Tests inject
  `set_remote_visibility_prober`; no test uses the network.
- Pass `sdd_secret_scanning: true` from `_sidecar_provider_options` in
  `src/sase/sdd/_sidecar_init.py` only for this role.
- Document that option on the `ws_preflight_sdd_sidecar` and `ws_create_sdd_remote`
  docstrings in `src/sase/workspace_provider/_hookspec.py`. sase-github already honors
  it.

Extend `tests/main/test_repo_init_handler_creation.py` and
`tests/test_linked_repo_sidecar_hidden_agents.py`.

## Step 4 — Bare clone, fetch URL, and read-side materialization

Generalize `ensure_attachments_private_bare_clone` in `src/sase/sdd/_sidecar_bare.py` so
the role is a parameter. Keep the private role's HTTP(S) refusal and its root commit
message `chore(attachments): initialize private store`.

For `attachments`:

- Accept the configured remote, including `file://` (the tests) and SSH.
- When `parse_hosted_git_remote` can derive an HTTPS URL, set `remote.origin.url` to
  that anonymous URL and `remote.origin.pushurl` to the configured remote. Otherwise set
  `origin` to the configured remote and leave `pushurl` unset.
- Root commit message: `chore(attachments): initialize public store`.
- Still a bare `--filter=blob:none` clone at
  `hidden_sidecar_clone_dir(project_key, "attachments")`.

Read-side auto-materialization is ungated. When discovery wants the public store, the
clone is missing, the configured remote exists, and `resolve_remote_visibility` returns
`public`, create the clone once per process with the bounded SDD network git timeout. A
failure leaves the store missing and does not raise into the reader. Callers then queue
`⇡ pending upload`. Tests inject the prober so a `file://` remote counts as public;
production `file://` URLs stay `unknown` without that injection.

## Step 5 — Role-parameterized `GitAttachmentStore`

`GitAttachmentStore.__init__` gains `name: str = "git"` and `layout: str = "private"`.
Existing callers stay the private git tier. `describe()` returns the label unchanged.

`discover` builds the public instance with `name="public"`, `layout="public"`, and label
`<owner/repo> (public)` from `describe_label`. The private instance keeps
`<owner/repo> (private)`.

Layout:

- Private `put` / `has` / `get` keep `artifact_object_relpath` (extensionless).
- Public `put` takes keyword-only `mime_type: str | None = None` and writes
  `attachment_public_object_relpath(sha256, mime_type)`. The `BlobStore` protocol
  signature stays as it is; only the git store and the upload path that holds the
  concrete store pass `mime_type`.
- Public `has` / `get` resolve the object by listing `files/objects/sha256/<xx>/` on the
  fetched tree (`git ls-tree`, no blob download) and accepting `<sha>` or `<sha>.<ext>`
  through `attachment_object_digest_from_relpath`. Tombstone paths stay extensionless.

`hidden_clone_path(project_key, role)` takes the role. Update every caller, including
`src/sase/bead/_sync_worker_run.py` and `src/sase/doctor/checks_attachment_store.py`.
`describe_label(clone, project_key, audience)` takes `public` or `private` and appends
that word once. The echo in `upload/echo.py` already avoids a second suffix; keep it
that way.

`discover_stores` returns the reachable stores under `public`, `git`, and `large`, in
that key order. A disabled role yields no store. A missing clone yields no store, after
the public role has had its one materialization attempt.

## Step 6 — Placement and outbox by audience

`pre_write_upload` already splits wires on `visibility == "public"`. Replace the
temporary "public always means missing store" branch:

| Audience                        | Tiers passed to `attachment_placement`                   | Outbox `store`   |
| ------------------------------- | -------------------------------------------------------- | ---------------- |
| public                          | `[public]` capped at `get_attachment_public_max_bytes()` | `public`         |
| private (and absent visibility) | `[git, large]` as today                                  | `git` or `large` |
| local-only                      | none                                                     | none             |

A public wire whose public store is missing is queued with `⇡ pending upload` and the
existing `sase repo init` hint. It is not placed on `git` or `large`. A private wire is
not placed on `public`. Over-cap public bytes have no next tier: they do not upload to
the public remote.

`post_write_queue` accepts placement `public` the same way it accepts `git`.
`_store_for_pending_item` rebuilds a public store with `name="public"` and
`layout="public"`. An item with `store_name="public"` and an empty `store_repo` resolves
through `discover_stores`, so a queue row written before the clone existed drains after
materialization.

Outbox (`src/sase/bead/attachments/outbox.py`):

- Identity is `(digest, store)`. Old rows with no extra fields stay valid (`store`
  already defaults to `git`). Keep `OUTBOX_SCHEMA_VERSION` at 1.
- Optional `mime_type` so a later public drain writes the right extension. When it is
  absent, the drain uses the descriptor MIME if a current note reference has one,
  otherwise the extensionless public path.
- Optional `state`: `pending` (default) or `blocked`. Drains skip `blocked`.
  `enqueue_outbox` dedups on `(digest, store)` and does not resurrect a blocked row as
  pending.
- `drain_outbox` uploads only entries whose `store` equals `store.name`. The sync worker
  drains `public`, then `git`, then `large`.
- Fetch's outbox map must tolerate two rows with one digest. A `blocked` row is not
  `pending_upload`. A pending row still is.

## Step 7 — Fetch misses and lazy discovery

`_ensure_discovered` copies `stores` as well as `project_key`, `store`, and `outbox`.

`attachment_state` / `_ensure_fetched` take the descriptor's effective visibility
(`public` or `private`; absent means private). Probe order:

- public: `public`, `git`, `large`
- private: `git`, `large`, `public`

Replace the size-based reorder with that order.

Split miss from digest mismatch. Add `missing: bool = False` on `BlobStoreError`.
`GitAttachmentStore.get` sets it when the tree has no object for the digest.
`RcloneAttachmentStore.get` sets it on its existing not-found path. In
`_ensure_fetched`:

- `missing` moves on to the next store.
- A digest mismatch (`mismatch` / `hash` in the message, or a non-missing non-transient
  error) records `corrupt` and stops.
- A transient error moves on, and the digest is `failed` only after every store has
  failed.

A public-store miss for a private-only object must not render `‼ digest mismatch`.

## Step 8 — `GH013`

In `sync_ref` (`git_store/plumbing.py`), classify push stderr before the `[rejected]`
retry branch. These are permanent:

- `GH013`
- `Push cannot contain secrets`
- `push declined due to repository rule violations`

Raise `BlobStoreError(..., transient=False)` with `secret_scan=True`. Do not return
`False` (that retries and would push the rejected commit again).

The upload / drain handler for `secret_scan`:

1. Reset the hidden clone's branch ref to the remote tip (fetched tip, or the pre-commit
   tip). The rejected commit must not ride a later push.
2. Mark that `(digest, store)` outbox entry `blocked`.
3. When the rejected store is `public` and a private store exists, `put` the bytes to
   the private store directly. Do not re-enter the public path.
4. Narrow every **note** descriptor that references the digest to `visibility: private`.
   Find them with the current-manifest reference walk (`lifecycle` inventory or
   `bead_attachment_references`). For each note, call `BeadProject.edit_note` with the
   note text unchanged and the manifest copied, changing only that descriptor, then
   publish through the same mutation path the CLI note editor uses. Leave `+1` evidence
   descriptors unchanged.
5. Echo `⛔ blocked by secret scanning — stored privately`.

A secret-scan rejection on the private store still resets the ref and marks that entry
blocked. It does not rewrite descriptors and does not upload anywhere else.

Tests simulate the rejection by stubbing the push result's stderr. Do not commit a
realistic secret literal. Build any token-shaped fixture by concatenation at runtime,
and prefer not to need one: the stub carries the `GH013` marker.

## Step 9 — Doctor and purge

`project.attachment_store` (`src/sase/doctor/checks_attachment_store.py`):

- Retitle to `Attachment stores` (the summary no longer says only "Private").
- Report each present store's reachability, tip, and outbox, including blocked entries.
  A disabled role is not a store.
- Skip when neither git store nor the large tier exists, as today.

Purge (`src/sase/bead/cli_attachment_lifecycle.py`) walks `public`, `git`, and `large`.
The scrub hint names the attachments repo for a public object and the
attachments-private repo for a private object. It never names the beads repo. Public
object paths in the hint use the extension form when the MIME has a canonical extension.

## Tests

Extend the existing modules (`tests/test_bead/test_git_attachment_store.py`,
`test_attachment_fetch.py`, `test_attachment_upload.py`, `test_attachment_lifecycle.py`,
`tests/sdd_store/test_sidecar_bare.py`, the repo-init and linked-repo tests) and add
`tests/test_bead/test_attachment_public_store.py` for the two-home cases.

Two homes are local `file://` bare remotes and two `SASE_HOME` directories. Inject the
visibility prober. No real GitHub remotes and no network.

Cover:

- Home A writes a public object; home B, with only the public remote and no private
  clone, reads it. The bytes match.
- A private object never appears on the public remote.
- A payload larger than `public_max_bytes` never appears on the public remote.
- Outbox drain uploads a `public` row only to the public store and a `git` row only to
  the private store. A missing public store leaves the public row queued.
- An old outbox row with only `digest` / `size_bytes` / `store: git` still drains to the
  private store.
- Simulated `GH013` walks the whole handler: ref reset, blocked row, private copy, note
  descriptor narrowed, `+1` descriptor unchanged, and the echo.
- Fetch: a miss on `public` followed by a hit on `git` returns the bytes and does not
  mark `corrupt`. A real digest mismatch still does.
- `_ensure_discovered` copies `stores`, so lazy discovery probes `large` and `public`.
- `disabled: true` hides each role in discovery and in the doctor even when the clone
  exists.
- Flag off: `sase repo init` does not offer the public repo. Flag off: discovery and
  read-side materialization still run.
- Flag on, non-public beads probe: creation is refused and the message names
  `repos.sidecar.builtin.beads.visibility: private`.
- Flag on, TTY: the public prompt contains `Repository visibility: PUBLIC.` and defaults
  to yes.

Every flag-dependent branch has an on test and an off test. Beads that attach nothing do
no new store work on the hot path (keep discovery lazy).

## Verify and close

Run `sase tool run check` in this checkout. A failure that reproduces the same way on
the clean base tree is a `PROPOSED FOLLOW-UP:` note citing any task bead that already
tracks it, and the phase still closes.

Before closing:

```bash
sase bead epic-symbols sase-1d5.4
```

There are no `--epic-symbol` entries today. If the change adds any, re-key the Justfile
line to the parent epic or to a later open phase (`sase-1d5.5` or `sase-1d5.6`).
`sase bead close` refuses while leftovers remain.

Then:

```bash
sase bead close sase-1d5.4 --note "<what sase tool run check and the two-home tests verified>"
```

Close only `sase-1d5.4`.
