---
tier: tale
title: Apply the sase-1ck.5 naming and --private fix
goal:
  sase-github creates sidecar repos with the configured visibility (including private),
  and epic sase-1ck's shared_store phase is re-scoped to a reserved attachments-private
  role so the plain attachments name stays free for a future public store.
size: small
proposed_by: bbugyi200.apollo.33
create_time: 2026-09-29 09:47:30
status: wip
---

<!-- sase:links:start -->

## Links

| Relation     | Artifact                                                                  | Why                                                                  |
| ------------ | ------------------------------------------------------------------------- | -------------------------------------------------------------------- |
| derives-from | [research:202609/bead_attachment_audience/bead_attachment_audience.md][1] | applies its section 8 sase-1ck.5 naming and --private recommendation |
| related      | [plan:202609/bead_note_attachments.md][2]                                 | amends the epic plan's shared_store phase naming                     |

[1]:
  https://github.com/sase-org/sase--research/blob/main/202609/bead_attachment_audience/bead_attachment_audience.md
[2]: https://github.com/sase-org/sase--plans/blob/main/202609/bead_note_attachments.md

<!-- sase:links:end -->

# Apply the `sase-1ck.5` naming and `--private` fix

## Context

Source: `research:202609/bead_attachment_audience/bead_attachment_audience.md`. Read it
with `sase artifact read` and look at §1 (the sase-github and sidecar-init rows), §8,
and §9, "Now" item 3. The report recommends that phase `sase-1ck.5` (`shared_store`) of
epic `sase-1ck` (bead note attachments):

1. name its reserved private role so that the plain `attachments` name stays free for a
   future public attachments store, and
2. add `--private` repository creation to sase-github, with a provider preflight that
   reports the requested visibility, and flip the test that forbids `--private`.

These facts were verified during planning:

- **The host already asks for private repos, but sase-github ignores the request.**
  `_sidecar_provider_options` in `src/sase/sdd/_sidecar_init.py` passes
  `options["sdd_visibility"]` to the provider. That value comes from
  `repos.sidecar.*.<role>.visibility`, a `public | private` schema enum. The provider
  hookspec (`src/sase/workspace_provider/_hookspec.py`) documents the option.
  `docs/init.md` and `docs/agents_sidecar.md` already tell users to set
  `visibility: private`. Meanwhile sase-github:
  - hardcodes `visibility="public"` in both returns of `preflight_sdd_sidecar`
    (`src/sase_github/workspace/sdd_sidecar.py`);
  - hardcodes `--public` in `create_github_sdd_repo`
    (`src/sase_github/workspace/sdd_repo.py`);
  - has a test that asserts `--private` is never passed
    (`tests/test_workspace_plugin.py`, `assert all("--private" not in cmd …)`).

  So today `preflight_sidecars` fails closed for **any** sidecar configured with
  `visibility: private`, including the documented agents-sidecar override. This is a
  live provider-contract defect, not only an attachments concern.

- **The role name becomes the repo suffix.** The host derives a sidecar repo as
  `<project>--<role>` and passes the role as `sdd_sidecar_suffix`. sase-github validates
  suffixes against `[a-z0-9][a-z0-9-]*`. The report writes the role as
  `attachments_private` but the repo as `<project>--attachments-private`, and an
  underscore role can't produce that repo. **Use the hyphenated role key
  `attachments-private`.** That makes role and suffix identical, like every existing
  role. Hyphens already work in `hidden_sidecar_clone_dir` and in the env-var role
  normalization in `src/sase/sdd/env.py`.
- **`sase-1ck.5` hasn't started.** It depends on `.4`, which is still in progress. The
  role doesn't exist in code yet: `RESERVED_SIDECAR_ROLES` holds only
  `plans | beads | agents`. The epic plan `plan:202609/bead_note_attachments.md` is the
  design file the phase worker reads through `sase bead read`. Its Background item 5,
  the Presentation write-echo example, and the Phase 5 section all name the role
  `attachments` / `sase-org/sase--attachments`. The epic plan validates cleanly today.

## Decision: code versus bookkeeping

- **The `--private` half is real code, landed now in sase-github.**
  - It stands alone.
  - It doesn't depend on any unlanded `sase-1ck` phase.
  - It fixes a defect that users can hit today.
  - It keeps the already-`large` `.5` phase from crossing into another repository.
- **The naming half is a scope amendment, not code.** There is no role to rename yet.
  Amend the epic's design of record (the plan file) and the `.5` bead, and add a dated
  bead note so the land agent can confirm it was addressed. Bead notes alone would leave
  the plan's `attachments` references contradicting the note that the phase worker also
  reads.
- **Out of scope.** Do not:
  - add any `attachments` or `attachments-private` role to sase code (that is `.5`'s
    work);
  - create any real GitHub repository;
  - give `.5` a public default or add a `visibility` wire field (the report says not to
    before the scanner/policy epic exists);
  - edit any other phase or bead;
  - handle the credential-rotation or push-protection items (separate work).

## Step 1 — sase-github honors `sdd_visibility`

Open the repo with `sase repo open sase-github -r "<why>"`, work only in the printed
path, and follow its `CLAUDE.md`.

- **`src/sase_github/workspace/sdd_repo.py`:**
  - Add `sdd_sidecar_visibility(options) -> str` next to `sdd_sidecar_suffix`.
    - A missing, `None`, or blank value returns `"public"`. This keeps legacy `sdd`
      storage and older hosts unchanged.
    - A string is normalized with `strip().casefold()` and must be `public` or
      `private`.
    - Anything else, including non-strings, raises
      `RuntimeError("unsupported SDD sidecar visibility: <repr>; expected public or private")`.
  - Give `create_github_sdd_repo` a keyword `visibility: str = "public"` and pass
    `f"--{visibility}"` where `--public` is hardcoded now. Keep the argument order
    unchanged otherwise.
- **`src/sase_github/workspace/sdd_sidecar.py`:**
  - `preflight_sdd_sidecar` resolves the visibility once, before any `gh` call, and
    reports it in both the `unavailable` return and the probe return.
  - `create_sdd_remote` resolves the visibility before probing and passes it to both
    `create_github_sdd_repo` calls: the exact `sdd_repo` target branch and the discovery
    branch. `materialize_sdd_store` inherits this through `create_sdd_remote`.
  - Existing (`found`) repos are adopted exactly as today. The hook contract defines
    visibility as creation intent, so don't verify an existing repo's visibility.
- **Tests (`tests/test_workspace_plugin.py`):**
  - Keep the default case: with no `sdd_visibility`, creation passes `--public` and
    never `--private`.
  - Add parametrized cases:
    - `sdd_visibility: "private"` runs `gh repo create <repo> --private --description …`
      and never `--public`. Cover the split-sidecar suffix path (for example suffix
      `attachments-private`) and the exact `sdd_repo` path.
    - An explicit `"public"` behaves as before.
    - `"PRIVATE"` is normalized.
    - `"internal"` and a non-string raise before any subprocess call.
    - Preflight reports `private` for both the `not_found` and `unavailable` paths.
      Extend the preflight tests near
      `test_preflight_reports_missing_sidecar_without_mutations`.
- **Docs:** `README.md` (the `sase sdd init` paragraph), `docs/configuration.md` (the
  managed-project sidecars paragraph), and `docs/architecture.md` (the creation-consent
  paragraph) should say that sidecars are created with the configured
  `repos.sidecar.*.visibility` (default public). Don't edit `CHANGELOG.md`;
  release-please owns it.
- **Verify** with `sase tool run check` inside the sase-github checkout.

## Step 2 — Amend the epic plan (design of record for `.5`)

Open the plans sidecar with `sase repo open plans -r "<why>"`. Edit
`202609/bead_note_attachments.md` at the printed path, after syncing so that you edit
the latest version. Change only the items below:

- **Frontmatter `phases[shared_store].description`:**
  `shared_store: add the reserved private attachments-private sidecar role (repo <project>--attachments-private, a hidden bare partial clone), the git BlobStore written with plumbing, and placement with explicit -L local-only. Add pre-publication uploads with an outbox fallback, capped lazy fetch, availability badges, attachment push, and a doctor check.`
  Keep valid YAML quoting.
- **Background item 5:** change "private `attachments` sidecar" to "private
  `attachments-private` sidecar".
- **Presentation write-echo example:** change both
  `sase-org/sase--attachments (private)` lines to
  `sase-org/sase--attachments-private (private)`.
- **`## Phase 5: shared_store`:**
  - Add a short `> Amendment (2026-09-29)` callout at the top that cites
    `research:202609/bead_attachment_audience/bead_attachment_audience.md` §8.
  - Rename the role bullet to `attachments-private`, with a module constant such as
    `ATTACHMENTS_PRIVATE_SIDECAR_ROLE`. Use
    `hidden_sidecar_clone_dir(project_key, "attachments-private")`, and change "Creating
    `sase-org/sase--attachments`" to `sase-org/sase--attachments-private`.
  - Add these bullets:
    - Don't reserve the plain `attachments` role. It is held for the follow-up
      public-attachments epic.
    - The role key is hyphenated because repos derive as `<project>--<role>` and the
      GitHub provider accepts only `[a-z0-9-]` suffixes.
    - sase-github now honors `sdd_visibility` (landed separately under this amendment),
      so this phase doesn't change sase-github. Add a sase-side `preflight_sidecars`
      test: a fake provider that reports `private` for a `visibility: private` role
      passes, and one that still reports `public` fails closed.
    - Keep the store private-only in this epic. Add no `visibility` descriptor field and
      no public default.
- Leave every other phase and section untouched.
- **Verify** with `sase plan validate <printed path>/202609/bead_note_attachments.md`,
  which must pass as it does today. Also grep the file to confirm that the only
  remaining plain-`attachments` role mentions are the reservation sentence and unrelated
  uses (config keys `bead.attachments.*`, CAS paths, and the `attachments` wire field).

## Step 3 — Bead bookkeeping for `sase-1ck.5`

- Run `sase bead update sase-1ck.5 -d "<the exact new frontmatter description>"`. Keep
  the `shared_store: ` prefix.
- Run `sase bead note sase-1ck.5 "<note>"` with a note that starts with
  `SCOPE AMENDMENT (2026-09-29, research:202609/bead_attachment_audience/bead_attachment_audience.md §8):`
  and states:
  - the role is now `attachments-private` (repo `<project>--attachments-private`), and
    the plain `attachments` name is reserved for a future public store;
  - sase-github `--private` creation and visibility-reporting preflight landed
    separately, so don't reimplement them; only add the sase-side preflight test;
  - the store stays private-only, and phases never create real GitHub repos;
  - the epic plan's Phase 5 carries the details.
- Run
  `sase bead ref add sase-1ck.5 research:202609/bead_attachment_audience/bead_attachment_audience.md`.
- Don't change any bead's status or assignee, and don't touch other beads.

## Acceptance

- sase-github `sase tool run check` passes. The new tests show:
  - private creation passes `--private`;
  - the default and explicit-public paths still pass `--public`;
  - invalid values fail before any `gh` call;
  - preflight reports the requested visibility.
- `sase plan validate` passes for the amended epic plan, and its Phase 5, Background,
  echo example, and `shared_store` frontmatter all name `attachments-private`.
- `sase bead read sase-1ck.5 -r "<why>"` shows the new description, the
  `SCOPE AMENDMENT` note, and the research ref.
- No tracked file in the sase primary repo changes. If one does, read the
  `lint_and_test` memory and run the required checks. The sase-github checkout and the
  plans sidecar become repository obligations in the final declaration.
