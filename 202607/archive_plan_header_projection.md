---
tier: tale
title: Project the canonical plan header when a plan enters the SDD archive
goal:
  Every plan that lands in the canonical SDD archive carries its derived header sections — the reciprocal `PROMPT` link,
  the rebased `PARENT` link, and the `BEAD` link — no matter which writer put it there, so `sase validate` stops failing
  on approved epic plans.
create_time: 2026-07-30 09:40:37
status: done
---

- **PROMPT:** [202607/prompts/archive_plan_header_projection.md](prompts/archive_plan_header_projection.md)
- **AGENTS:**
  - [bbugyi200.athena.pc--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.pc.md#member-code)
  - [bbugyi200.athena.pc--plan](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.pc.md#member-plan)
- **COMMITS:**
  - [4866ece](https://github.com/sase-org/sase/commit/4866ece4a6dca7bc148f10323b5ce418f145729d) — fix(sdd): project
    canonical plan header links

# Project the canonical plan header when a plan enters the SDD archive

## Problem

`just check` / CI fails deterministically in `sase validate`:

```
  fail   plan links validate

error: 202607/prompts/vcs_backed_artifact_capture.md: 202607/vcs_backed_artifact_capture.md is missing a valid 'prompt' link (reverse-link)
error: 202607/vcs_backed_artifact_capture.md: missing 'prompt' link to 202607/prompts/vcs_backed_artifact_capture.md (missing-link)
```

This is **not** a transient failure. It reproduces on a clean checkout of the plans sidecar with
`sase plan links validate`, and it will reproduce on every future run until the plan file is repaired, because the
broken state is committed data (`sase--plans` commit `87e25eae`, "Archive approved plan vcs_backed_artifact_capture").
It will also recur for every future approved epic until the code is fixed.

## Root cause

Two functions write plans into the canonical SDD plan archive, and only one of them applies the derived plan-header
projection.

- `write_sdd_files()` (`src/sase/sdd/_write.py:60`) formats, stamps `create_time`, normalizes `tier`, and then applies
  the full projection: `update_source_aware_artifact_link(..., SddArtifactLinkType.PROMPT, ...)` at
  `src/sase/sdd/_write.py:100`, `refresh_existing_parent_section()` at `:113`, and `refresh_bead_plan_section()` at
  `:120`.
- `archive_plan_file()` (`src/sase/sdd/plan_archive.py:51`) formats, stamps `create_time`, normalizes `tier`, and
  validates — and applies **none** of the three projections. Its own docstring claims to centralize archive preparation
  "so approval surfaces and `sase bead work <plan-file>` cannot drift", which is exactly the invariant it breaks.

Approved **epic** plans reach the archive only through the second path:

1. The planning agent's accept step takes the `is_epic` branch at `src/sase/axe/run_agent_exec_plan_accept.py:334` and
   calls `write_sdd_spec()` (`src/sase/sdd/_write.py:26`). That writes the prompt snapshot with its `- **PLAN:**` bullet
   and, by design, does not write the plan (`src/sase/axe/run_agent_exec_plan_accept.py:387-388`: "Epic prompt snapshots
   are committed independently"). Commit: `478d0812` "Add SDD prompt for …".
2. The `epic` approval choice carries `archive_side_effect=False` (`src/sase/plan_approval_choices.py:254`), so
   `run_plan_side_effects()` never archives it. The archive happens later inside `sase bead work`, at
   `src/sase/bead/cli_work_from_plan.py:231` → `archive_plan_file(..., tier="epic")`. Commit: `87e25eae` "Archive
   approved plan …", 436 insertions, no `- **PROMPT:**` bullet.

So the prompt→plan direction is written and the plan→prompt direction never is. `validate_sdd_tree()` reports that as
one `missing-link` error on the plan (`src/sase/sdd/_link_validation.py:83-91`) plus one `reverse-link` error on the
prompt (`:206-219`).

`archive_plan_file()` skipping `refresh_existing_parent_section()` is the same defect in a second dimension: an epic
plan whose `PARENT` bullet was authored relative to the planner's durable artifact is never rebased for its archive
destination, which surfaces later as a `parent-missing-target` error from `src/sase/sdd/_link_validation.py:294-327`.

## Why tale plans currently look healthy

Tale plans go through both writers, and correctness today is an accident of ordering. The `approve` and `commit` choices
do set `archive_side_effect=True` (`src/sase/plan_approval_choices.py:68,85`), so `_archive_plan_for_approval()`
(`src/sase/plan_approval_actions.py:430`) archives the tale from the planner's durable artifact with
`preserve_existing=False` — i.e. it overwrites the sidecar plan with content that has no `PROMPT` bullet. Tales survive
only because the agent's `write_sdd_files()` write lands afterwards and wins. There are 140 "Archive approved plan"
commits in `sase--plans`, including recent tale names, so this ordering is exercised constantly. Fixing
`archive_plan_file()` removes the latent flake as well as the epic bug.

## Constraint that shapes the fix

The `PROMPT` bullet must **not** be installed unconditionally. In `202607` alone, 48 archived plans have no prompt
snapshot at all (they are `_1.md` duplicates or came from the `approve` choice, whose `commit_plan=False` means the
prompt snapshot is never committed). Those are `unpaired-file` **warnings** today
(`src/sase/sdd/_link_validation.py:102-110`). An unconditional install would point them at a nonexistent file and
convert 48 warnings into `link-missing-target` **errors** (`:173-181`) — trading one CI failure for dozens. The install
must therefore be driven by a resolved prompt path, not assumed.

## Design

Add one shared projection helper and route all three archive-adjacent write points through it. Existence policy lives at
the call sites, which know whether a prompt snapshot is guaranteed; the helper only installs what it is handed.

Labels stay canonical without extra work: for `sidecar_repos` storage, `store.sdd_dir`, `store.kind_root("plans")`, and
`resolve_sdd_root(None)` are all the same directory, so `stable_sdd_reference()` yields `202607/prompts/<name>.md` —
byte-identical to what `sase plan links repair` considers current.

No `sase-core` change is required. This composes existing Rust-owned primitives (`sdd_plan_header_block_upsert_section`
via `upsert_plan_header_section()`); the wire contract is unchanged.

## Changes

### 1. `src/sase/sdd/plan_header_writes.py` — the shared projection

Add two functions and export both.

```python
def archived_prompt_path_for_plan(plan_path: Path) -> Path:
    """Return the canonical prompt snapshot path paired with *plan_path*."""
    return plan_path.parent / "prompts" / plan_path.name


def project_plan_header_sections(
    document: str,
    *,
    sdd_dir: Path,
    plan_path: Path,
    plans_root: Path,
    prompt_path: Path | None,
    store: SddStore | None = None,
    primary_root: Path | None = None,
) -> str:
    """Apply every derived header section a canonical plan owns."""
```

`project_plan_header_sections()` applies, in this order:

1. When `prompt_path is not None`,
   `update_source_aware_artifact_link(document, sdd_dir, plan_path, prompt_path, SddArtifactLinkType.PROMPT, remove_legacy=True)`.
   When it is `None`, leave any existing `PROMPT` section untouched — never remove one, so a plan whose snapshot is
   merely absent from a stale checkout is not silently unlinked.
2. `refresh_existing_parent_section(..., source_path=plan_path, plans_root=plans_root, store=store, primary_root=primary_root)`.
3. `refresh_bead_plan_section(..., store=store, primary_root=primary_root)`.

`plan_path` must be the plan's **destination** path, not its source, so the computed hrefs are relative to where the
file will live.

Note the import direction: `plan_header_writes.py` already imports from `plan_header_block`, and `artifact_links.py`
imports from `plan_header_block` too — import `artifact_links` lazily inside the function body if a cycle appears,
matching the lazy-import style already used in this module.

### 2. `src/sase/sdd/plan_archive.py` — apply the projection

`archive_plan_file()` gains two optional keywords, `primary_root: Path | None = None` and
`prompt_path: Path | None = None`, and applies the projection between tier normalization and validation:

```python
content = set_frontmatter_fields(content, {"tier": tier})
content = project_plan_header_sections(
    content,
    sdd_dir=store.sdd_dir,
    plan_path=destination,
    plans_root=plans_root,
    prompt_path=_resolved_prompt_path(destination, prompt_path),
    store=store,
    primary_root=primary_root or resolve_checkout_anchor().root,
)
content = format_with_prettier(content)
validate_plan_for_commit(content, tier=tier, path=destination, yyyymm=archive_month)
```

`_resolved_prompt_path()` returns an explicitly passed `prompt_path` when given, otherwise
`archived_prompt_path_for_plan(destination)` if that file exists and `None` if it does not.

Use `resolve_checkout_anchor().root` (`src/sase/sdd/checkout_anchor.py:18`) rather than bare `Path.cwd()`, consistent
with the anchoring fix in `5ba1f08d0`. The second `format_with_prettier()` call is new and mirrors
`src/sase/sdd/_write.py:125`; it keeps the rendered bullet block prettier-stable before validation.

The early-return contract is unchanged: a source already under `plans_root` is still a no-op, and `preserve_existing`
still short-circuits before any content work.

### 3. `src/sase/bead/epic_from_plan.py` — reproject at the bead-link write

Lines 155-161 currently call `refresh_bead_plan_section()` alone. Replace that with `project_plan_header_sections()`,
passing the same `store` and `primary_root` it already threads (supplied as `workspace_dir` from
`src/sase/bead/cli_work_from_plan.py:366`) and letting `prompt_path` resolve by existence.

This is the second-chance repair that closes the remaining race. `archive_plan_file()` can legitimately see no prompt
snapshot when `_pull_sdd_clone()` (`src/sase/sdd/_store_link.py:267`) is inside its failed-integration cooldown and the
sidecar checkout is stale. The bead-link write happens later in the same `sase bead work` run, after
`_require_plan_store_health()` and inside `store_git_write_lock`, so by then the snapshot is present and the `PROMPT`
bullet gets installed on the commit that already touches the plan.

### 4. `src/sase/sdd/_write.py` — route `write_sdd_files()` through the shared helper

Replace the three separate projection calls at `:100-124` with one `project_plan_header_sections()` call.

**This step has a trap.** `write_sdd_files()` computes `plan_content` _before_ calling `write_sdd_spec()` at `:133`, so
the prompt file does not exist yet at projection time and an existence probe would skip the install. Pass the
already-known `prompt_path` explicitly (`prompt_path=prompt_path`, computed at `:85`) so the install is unconditional
here. Do **not** reorder `write_sdd_spec()` earlier: today `validate_plan_for_commit()` raises before any file is
written, and reordering would leave an orphan prompt snapshot on validation failure.

Behavior must be byte-identical for tale writes. Verify with the existing tests before and after.

## Data repair to unblock CI

Code alone does not fix the committed breakage. Repair the one broken pair in the `sase--plans` sidecar.

Open the sidecar through the skill (`sase repo open sase--plans -r "..."`) and use the path it prints. Then, from the
project workspace:

```bash
sase plan links repair --path <printed-plans-path>          # dry run; confirm the actions
sase plan links repair --path <printed-plans-path> --write
```

The dry run currently reports exactly three actions:

| plan                                    | current                           | note                                 |
| --------------------------------------- | --------------------------------- | ------------------------------------ |
| `202607/vcs_backed_artifact_capture.md` | `null`                            | the actual breakage                  |
| `202607/artifact_read_cli.md`           | `prompts/artifact_read_cli.md`    | label drift only; resolves, no error |
| `202607/preview_panel_reader.md`        | `prompts/preview_panel_reader.md` | label drift only; resolves, no error |

Taking all three is fine and desirable — the two drifted labels are pre-existing non-canonical shapes from an older
storage layout and normalizing them makes the tree agree with the repair tool. If a minimal diff is preferred, install
only the `vcs_backed_artifact_capture` bullet; that alone clears both errors.

Commit the sidecar change via the normal sase commit workflow. Confirm with `sase plan links validate`, which must
report `0 errors` (warnings are expected and unchanged).

## Tests

New tests, following the existing split-by-concern convention in `tests/`:

1. `archive_plan_file()` installs the reciprocal `PROMPT` bullet when the prompt snapshot exists at
   `<month>/prompts/<name>.md`, with the canonical label `<yyyymm>/prompts/<name>.md` and a `prompts/<name>.md` href.
2. `archive_plan_file()` leaves the plan unlinked when no prompt snapshot exists — the regression guard for the 48
   legitimately unpaired plans. Assert no `PROMPT` section, and that `validate_sdd_tree()` yields an `unpaired-file`
   warning rather than a `link-missing-target` error.
3. `archive_plan_file()` does not remove a pre-existing `PROMPT` bullet when the snapshot is absent from the checkout.
4. `archive_plan_file()` rebases an authored `PARENT` bullet for the archive destination.
5. `archive_plan_file()` preserves the `BEAD` bullet for a plan carrying `bead_id`, and orders sections `PROMPT` before
   `PARENT` before `BEAD` (the Rust-owned fixed order).
6. End-to-end pairing: archive a plan next to a prompt snapshot that already links back, then assert
   `validate_sdd_tree()` reports zero errors — the direct regression test for this CI failure.
7. `archive_plan_file()` remains a no-op for a source already under `plans_root` and under `preserve_existing=True`.
8. `write_sdd_files()` output is unchanged, including that the `PROMPT` bullet is installed even though the prompt file
   does not exist when the plan content is computed.
9. `project_plan_header_sections()` unit tests for `prompt_path=None` vs a supplied path.

Existing suites that must stay green: `tests/test_sdd_plan_tiers.py`, `tests/main/test_plan_links_validate_handler.py`,
`tests/main/test_validate_handler.py`, and the plan-archive and `bead work` tests.

## Verification

```bash
just install
just lint
just test
sase plan links validate      # 0 errors
just check
```

`just install` first — the workspace virtualenv may be stale.

## Out of scope

- The 537 pre-existing warnings (legacy frontmatter links, unpaired plans, `_1.md` duplicates). Not errors; not this
  change.
- Backfilling prompt snapshots for the 48 unpaired `202607` plans.
- `sase validate` does not run `plan links repair --check`, and this change does not add it. Worth considering
  separately, but a repair gate on a shared sidecar is its own design question.
- Any `sase-core` / `sase_core_rs` change. The header-block contract is already correct and is reused as-is.
