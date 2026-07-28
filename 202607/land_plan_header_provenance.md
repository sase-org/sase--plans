---
tier: tale
title: Land the plan-header provenance epic on a published core floor
goal:
  Every behavior epic sase-ag added exists in the published sase-core-rs wheel this repo requires, a CI guard fails when
  it does not, and sase-ag is closed.
bead: sase-ag
create_time: 2026-07-28 13:12:58
status: done
---

- **PROMPT:** [202607/prompts/land_plan_header_provenance.md](prompts/land_plan_header_provenance.md)
- **PARENT:** [202607/plan_header_provenance.md](https://github.com/sase-org/sase--plans/blob/main/202607/plan_header_provenance.md)
- **BEAD:** [sase-ag](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ag/README.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-ag.land--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-ag.land.md#member-code)
- **COMMITS:**
  - [702f1ae](https://github.com/sase-org/sase/commit/702f1aece2375113427d437497924e960d5ca735) — build(deps): require sase-core-rs 0.12.4 (sase-ag)

# Plan: Land the plan-header provenance epic on a published core floor

## Goal

Epic `sase-ag` is functionally complete and `just check` is green, but it landed two Rust changes in `../sase-core` that
no published `sase-core-rs` wheel contains. Publish them, raise this repo's declared floor to that wheel, guard the two
behaviors that make it the true minimum, and then close `sase-ag`.

## Why this is the epic's own work

`sase-ag` phases 4 and 5 committed to `sase-core` after the `v0.12.3` release:

- `22fb5ba fix(plan): ignore fenced header examples (sase-ag.4)`
- `4d70c1c feat(plan): support canonical parent header migration (sase-ag.5)`

`pyproject.toml` still declares `sase-core-rs>=0.12.3,<0.13.0`. Local `just install` builds the extension from the
linked checkout, so every local gate passes and hides the gap. Against the declared published minimum it is real and
reproducible:

```
$ uv pip install --python /tmp/core0123/bin/python "sase-core-rs==0.12.3"
$ /tmp/core0123/bin/python -c '...sdd_plan_header_block_parse(plan_header_provenance.md)...'
0.12.3 disposition: invalid | reason: discontiguous or nested plan header bullets found
0.12.3 legacy parent removed: False
```

Consequences on a published install:

1. `sase plan links validate` — a `just validate` and CI gate — errors on `plans:202607/plan_header_provenance.md`,
   because its fenced Markdown example of the header block is read as a second, discontiguous live header. This is
   exactly the failure `sase-ag.1` and `sase-ag.3` recorded in their bead notes before `22fb5ba` landed. Plan viewers
   degrade to no provenance rows for the same reason.
2. `sase plan links refresh --write` cannot finish the `parent:` migration the epic promises, because
   `sdd_plan_header_block_upsert_section` does not strip the legacy top-level key.
3. `tests/test_plan_validate.py:109` asserts the `parent-frontmatter-deprecated` diagnostic, which only exists in the
   unreleased core. The repo's declared minimum cannot pass the repo's own suite.

Raising the floor as part of landing Rust-core work is this repo's established convention, and the immediately preceding
land agent did it for this very epic's first phase:

- `c9978edf4 build(deps): require sase-core-rs 0.12.2 (sase-af.3)`
- `a643a864c fix(sdd): keep published core integration commit-safe (sase-af)` — "Raise the published core floor to the
  first wheel that exports both the lumberjack validator and the complete plan-header binding family."

That commit's guard (`tools/smoke_sase_core_rs_plan_header`) checks only that the six plan-header bindings **exist**.
Both gaps above are behavioral, so the existing guard passes against `0.12.3` and must be extended.

## Implementation

### 1. Publish `sase-core` v0.12.4

`sase-org/sase-core` PR **#41** (`chore: release v0.12.4`, branch `release-plz-2026-07-28T15-46-01Z`) is
release-plz-generated, `MERGEABLE`, `mergeStateStatus: CLEAN`, and all checks pass. Its changelog covers exactly the two
commits above and nothing else.

Merging publishes a release and a PyPI wheel, so it is outward-facing: it is gated on this plan's approval and must not
be done any other way.

```bash
gh pr merge 41 --repo sase-org/sase-core --squash
```

Then wait for the wheel to be installable before touching `pyproject.toml`:

```bash
uv pip install --python /tmp/core-min/bin/python "sase-core-rs==0.12.4"
```

If the release job has not published yet, poll rather than lowering the floor. If it fails outright, stop and report; do
not close `sase-ag` on an unreleased core.

### 2. Raise the declared floor

- `pyproject.toml`: `sase-core-rs>=0.12.4,<0.13.0`.
- Refresh `uv.lock` the way `a643a864c` did (it updated the lock alongside the bump).
- `tests/test_sase_core_rs_telemetry_smoke_tool.py::test_declared_minimum_tracks_pyproject_dependency` pins the expected
  minimum as a literal; update `"0.12.3"` to `"0.12.4"`.
- `just validate` runs `tools/validate_sase_core_rs_version --pyproject pyproject.toml --published-minimum`, which
  checks the floor against PyPI. It passes only once step 1 has actually published.

### 3. Make the floor behaviorally enforced, not just binding-shaped

Extend `validate_round_trip` in `tools/smoke_sase_core_rs_plan_header` with the two behaviors that distinguish `0.12.4`
from `0.12.3`, so a future editable linked-core build cannot mask a behavioral regression the way it masked this one:

- **Fenced examples are not live headers.** Parse a document whose body contains a fenced ` ```markdown ` block
  containing `- **PROMPT:**` and `- **AGENTS:**` bullets. Assert the disposition is `canonical` (it is `invalid` with
  reason `discontiguous or nested plan header bullets found` on `0.12.3`) and that the fenced block survives verbatim in
  the returned body.
- **Legacy `parent:` is removed on upsert.** Upsert a `PARENT` section into a document whose frontmatter carries a
  top-level `parent:` key, with `remove_legacy=True`. Assert the top-level key is gone and that unrelated frontmatter,
  including nested YAML, is preserved byte-for-byte.

Add both results to the dict `validate_round_trip` returns and update
`tests/test_sase_core_rs_plan_header_smoke_tool.py::test_installed_core_plan_header_round_trip`, which asserts that dict
exactly. Keep `test_plan_header_smoke_requires_complete_binding_family` working.

Prove the guard actually bites before trusting it: run the extended tool against a `0.12.3` venv and confirm it now
fails, then against `0.12.4` and confirm it passes.

```bash
uv venv --python 3.12 /tmp/core-0123 && uv pip install --python /tmp/core-0123/bin/python "sase-core-rs==0.12.3"
/tmp/core-0123/bin/python tools/smoke_sase_core_rs_plan_header   # must FAIL
/tmp/core-min/bin/python tools/smoke_sase_core_rs_plan_header    # must PASS
```

`.github/workflows/ci.yml` already runs this tool in the `published-core-minimum-smoke` lane; no workflow edit is
needed.

### 4. Commit

One commit in this repo, attributed to `sase-ag`. No `sase-core` source change is needed beyond merging #41.

### 5. Land the epic

This is the final step of the epic and must run after step 4, in this order:

1. `sase bead close sase-ag`. All six phase beads are already `CLOSED` with `resolution: done`, so the close should be
   accepted. If it is rejected, finish the named phases — never `--force` merely to make the command succeed.

2. `just symvision`. The four `sase-ag` epic-symbol whitelist entries in `Justfile` (lines 239-242) expire at close.
   Remove all four lines, then resolve what symvision reports. Simulated post-close, it reports exactly:

   ```
   Unused public functions/classes...
     PlanHeaderDocument in src/sase/sdd/plan_header_block.py
     PlanHeaderLegacyLink in src/sase/sdd/plan_header_block.py
     remove_plan_header_section in src/sase/sdd/plan_header_block.py
     replace_plan_header_block in src/sase/sdd/plan_header_block.py
   ```

   Follow the decision hierarchy in `sase/memory/symvision.md` (read it first; test references never keep a public
   symbol alive):
   - `replace_plan_header_block` and `remove_plan_header_section` have no non-test consumer anywhere. The five
     production consumers of the adapter (`artifact_links.py`, `plan_header_writes.py`, `plan_links_refresh.py`,
     `plan_display.py`, `_link_validation.py`) use only `parse_plan_header_block`, `render_plan_header_block`, and
     `upsert_plan_header_section`. Delete both wrappers, their `__all__` entries, and their tests in
     `tests/test_plan_header_block.py`. The Rust entry points and their PyO3 bindings stay: the block contract still
     owes upsert/replace/remove, and `tools/smoke_sase_core_rs_plan_header` exercises `sdd_plan_header_block_replace`
     and `sdd_plan_header_block_remove_section` directly against the published minimum, so published-API coverage is
     unaffected.
   - `PlanHeaderDocument` and `PlanHeaderLegacyLink` cannot be deleted — `parse_plan_header_block` returns the first and
     constructs the second — and no file imports either by name, so rule 2 applies: make them private
     (`_PlanHeaderDocument`, `_PlanHeaderLegacyLink`), update in-file references and `__all__`, and keep
     `parse_plan_header_block` public. Their public sibling types `PlanHeaderSection`, `PlanHeaderSectionKind`, and
     `PlanHeaderEntry` have real cross-file consumers in `plan_header_writes.py` and must stay public.

   Do not add a pragma: `tools/` does not reference these Python symbols, only the Rust binding names.

3. Set `status: done` in the frontmatter of `plans:202607/plan_header_provenance.md`
   (`sase/repos/plans/202607/plan_header_provenance.md`, currently `status: wip`).

## Verification

- `just install` first — this workspace is an ephemeral clone and `pyproject.toml` changes in step 2.
- `just check` after the code changes and again after the symvision cleanup in step 5.
- `just validate` must pass; it exercises both `validate_sase_core_rs_version --published-minimum` and
  `sase plan links validate` across all 3,250 committed plans.
- The `0.12.3`-fails / `0.12.4`-passes pair from step 3 is the proof that the new floor is the true minimum. Record both
  outcomes in the bead note.

## Out of scope

- `sase plan links refresh --write`. A dry run currently reports 17 plans that would change, all of them normal
  projection drift: agent artifact metadata and commits land after the plan file was last written, and the epic designed
  `reconcile` as the repair path for exactly that. Running it would commit and push ~17 plan files to the `sase--plans`
  sidecar, which is unrelated to landing this epic.
- The epic plan's own empty `COMMITS` section. `SASE_PLAN=` footers are written from the launcher environment, and epic
  phase agents run with `SASE_EPIC_PLAN_REF` instead, so no `sase-ag.N` commit carries a `SASE_PLAN` tag and the roll-up
  has no descendant tale plans to gather. That is the sourcing rule the epic plan specified, not a regression introduced
  by any phase. Broadening commit attribution to epic phases is a separate feature.
