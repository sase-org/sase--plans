---
tier: tale
title: Land bead creator attribution (epic sase-bv)
goal:
  "`sase bead create` records the acting SASE agent as the bead's creator instead of silently falling back to the store
  owner, the superseded `SASE_AGENT` guard in the attribution resolver is gone, and epic `sase-bv` is closed with its
  plan file marked done."
proposed_by: bbugyi200.athena.sase-bv.land
bead: sase-bv
create_time: 2026-07-31 11:08:20
status: done
---

- **PROMPT:** [202607/prompts/land_bead_creator_attribution.md](prompts/land_bead_creator_attribution.md)
- **PARENT:**
  [202607/bead_created_by_attribution.md](https://github.com/sase-org/sase--plans/blob/main/202607/bead_created_by_attribution.md)
- **BEAD:** [sase-bv](https://github.com/sase-org/sase--beads/blob/main/pages/sase-bv/README.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-bv.land](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-bv.land.md)
- **COMMITS:**
  - [4fd54a9](https://github.com/sase-org/sase/commit/4fd54a96707a0931b3c86a8c3551a2e0fbed0ea5) — fix(bead): route sase
    bead create through the attributing handler
  - [3a98c68](https://github.com/sase-org/sase/commit/3a98c68df821b90d4445fbb5bff8c132fb42757c) — refactor(bead): remove
    superseded SASE_AGENT guard from attribution

# Plan: Land bead creator attribution (epic sase-bv)

## Context

Epic bead `sase-bv` ("Attribute beads to the agent that created them") closed all five of its phases, and four of the
five landed correctly. Verification of the fifth uncovered one defect that voids the epic's own stated done-criterion,
plus one piece of integration debt against a change that landed while the epic was in flight.

### What the epic already landed correctly

- `sase-bv.1` (sase-core): `BeadCreateRequestWire.created_by: Option<String>` plus the request → phase-parent →
  store-owner resolution order in `create_issue`, and the system-managed `proposed_by` plan frontmatter field.
- `sase-bv.2`: `src/sase/bead/attribution.py` (`acting_agent_name`, `plan_proposed_by`, `resolve_bead_creator`),
  `created_by` threaded through `sase.core.bead_mutation_facade.create` and `BeadProject.create`, and `proposed_by` on
  the Python validated-plan dataclass.
- `sase-bv.3`: `proposed_by` stamped by `sase plan propose`, and a resolved creator passed from `handle_bead_create` and
  from `create_and_launch_epic_from_plan`.
- `sase-bv.4`: the `CREATED BY` block in `render_issue_detail`, `resolve_bead_creator_url`, `created_by_url` in the
  detail JSON envelope, refreshed goldens, and `docs/beads.md`.
- `sase-bv.5`: the linked `Created by` fact on published bead pages.

All of the above was read and confirmed present in the source. `just symvision` is clean (the epic's temporary
`--epic-symbol` whitelist entries were already removed by `sase-bv.3`) and `sase validate` passes all four checks.

### Defect 1 — `sase bead create` never reaches the Python attribution

The epic plan asserted that all bead creation funnels through `BeadProject.create`. It does not. `sase`'s entry point
short-circuits every `bead` invocation through a Rust fast path before argparse ever runs:

```python
# src/sase/main/entry.py
if len(sys.argv) >= 2 and sys.argv[1] == "bead":
    from .bead_fast_path import try_handle_bead_fast_path

    exit_code = try_handle_bead_fast_path(sys.argv[2:])
    if exit_code is not None:
        sys.exit(exit_code)
```

`try_handle_bead_fast_path` in `src/sase/main/bead_fast_path.py` declines only `close`, `list`, `show`, and
`search --format full`. Everything else — including `create` — is handed to the `bead_cli_execute` Rust binding, which
builds its request in `crates/sase_core/src/bead/cli.rs`:

```rust
let request = BeadCreateRequestWire {
    title: parsed.title,
    issue_type: parsed.issue_type,
    // ... no created_by ...
    ..BeadCreateRequestWire::default()
};
```

`created_by` therefore defaults to `None`, and `create_issue` falls through to the store owner. The creator resolution
added to `handle_bead_create` by `sase-bv.3` is unreachable in production: the handler only runs when the fast path
declines, which it does not for `create`.

This was confirmed empirically against a build of the current tree. In a throwaway git repo with a fresh bead store:

```text
$ SASE_AGENT_NAME=q8 sase bead create -T task -t "probe" -d "probe"
Created task: beads-1 — probe
$ # issues.jsonl
{"id": "beads-1", "owner": "", "created_by": ""}
```

The epic's stated completion criterion — "from a SASE agent, `sase bead create -T task -t '…'` records that agent's
global name" — is false today.

The reason `sase-bv.3`'s tests did not catch it is that every bead-create test calls `bead_cli.handle_bead_create(args)`
directly (see `tests/test_bead/test_task_beads.py`), so no test exercises the real `main()` dispatch. The epic-from-plan
path (`create_and_launch_epic_from_plan`) is pure Python and is genuinely correct; only the `sase bead create` CLI path
is broken.

### Defect 2 — dead `SASE_AGENT` branch in the attribution resolver

`sase-bv.2` defended against a hazard the epic plan documented and deliberately left to bead `sase-bp`: at plan time,
`discover_agent_identity` treated the bare `SASE_AGENT` launcher flag (`SASE_AGENT=1`) as a _name_ source, so a naive
resolver would attribute beads to an agent literally named `1`. `attribution.acting_agent_name` guards against it:

```python
_UNTRUSTED_IDENTITY_SOURCES = frozenset({"SASE_AGENT"})
...
        if identity.source in _UNTRUSTED_IDENTITY_SOURCES:
            meta_name = agent_name_from_meta(identity.artifacts_dir)
            if meta_name is None:
                return None
            name = meta_name
```

Bead `sase-bp` landed as commit `14d66229a` ("fix(agent): ignore run marker during identity discovery") _before_
`sase-bv.2` was committed, and it fixed the hazard at the source: `AgentIdentitySource` is now
`Literal["SASE_AGENT_NAME", "agent_meta"]`, and `discover_agent_identity` skips the marker and falls through to
`agent_meta.json` on its own. `identity.source` can never equal `"SASE_AGENT"` any more, so the constant, the branch,
and the `agent_name_from_meta` import exist only to service an impossible case. This is exactly the duplication the
landing pass is meant to remove.

### Verified fix for defect 1

Declining `create` in the fast path, exactly as `close` is already declined, restores correct attribution. Confirmed
against a build of the tree with that one change applied:

```text
$ SASE_AGENT_NAME=q8 sase bead create -T task -t "probe" -d "probe"
Created task: beads-1 — probe
$ # issues.jsonl
{"id": "beads-1", "owner": "", "created_by": "bbugyi200.athena.q8"}
```

This is the right shape for three reasons:

1. There is direct precedent. `close` already declines with a comment explaining that the Rust fast path cannot produce
   the information the Python handler owns; creator resolution is the same kind of host-side concern.
2. The alternative — threading `created_by` through the `bead_cli_execute` binding — is a cross-repo wire change, and
   the fast path only has raw `argv`, so resolving a `plan(<file>)` bead's `proposed_by` would force it to re-implement
   `parse_type_arg` outside the parser that owns it.
3. `handle_bead_create` is not a vestigial path. It is the implementation almost every bead-create test already
   exercises (`test_task_beads.py`, `test_cli_changespec.py`, `test_cli_auto_commit.py`), covering tier, size, refs,
   ChangeSpec metadata, parent validation, and the auto-commit message.

The cost is latency: measured on the same machine, `sase bead create` goes from ~0.42 s to ~0.71 s, matching what
`sase bead close` and `sase bead show` already pay. Bead creation is an occasional interactive command, not a hot loop,
so this is an acceptable trade for correct provenance.

## Design

Two independent, small changes plus the regression test whose absence let defect 1 ship, then the epic landing.

### `create` leaves the fast path

`try_handle_bead_fast_path` grows `create` into the same early-decline branch that already holds `close`, with the
comment extended to say why. Nothing else in `bead_fast_path.py` changes: `execute_bead_cli` keeps handling `create` for
any caller that asks for it directly, and `_MUTATING_VERBS` keeps listing `create` so the write-sandbox guard still
fires for those callers.

### The attribution resolver stops defending a fixed hazard

`acting_agent_name` reduces to discover → validate → globalize. The `_UNTRUSTED_IDENTITY_SOURCES` constant, the fallback
branch, and the now-unused `agent_name_from_meta` import are removed, and the module comment is rewritten to record that
`discover_agent_identity` itself ignores the `SASE_AGENT` marker. The never-raises contract, and the
`except Exception: return None` that implements it, are unchanged.

The four `acting_agent_name` tests that set `SASE_AGENT=1` stay exactly as they are. They assert observable behavior —
the marker is never mistaken for a name, and `agent_meta.json` still wins — which is still true and is now guaranteed by
the shared helper. They are the regression net proving the removal is safe.

### Non-goals

- No change to the Rust core, to the `bead_cli_execute` binding, or to any wire type. Defect 1 is fixed entirely in this
  repo.
- No `--created-by` flag. The value stays derived, per the epic's design.
- No backfill of `created_by` on existing beads.
- No change to dependency-edge or note attribution (`add_dependency`, `append_issue_note` still record the store owner).
  Those are outside the epic's scope; do not expand into them.

## Phase 1 — Route `sase bead create` through the attributing handler

**`src/sase/main/bead_fast_path.py`** — in `try_handle_bead_fast_path`, replace the `close`-only decline with a decline
for both verbs, and extend the existing comment so the new entry documents itself:

```python
    # The Rust close fast path does not yet expose the classification fields
    # needed for truthful close/already-closed/noted/cascade rendering, and the
    # Rust create fast path cannot resolve the acting SASE agent that
    # ``handle_bead_create`` records as the bead's creator.
    if argv[0] in {"close", "create"}:
        return None
```

**Update the four tests that use `create` as their representative fast-pathed verb.** All four fail with the change and
all four are testing something other than `create` itself; retarget each at a verb that is still fast-pathed rather than
weakening the assertion:

- `tests/main/test_bead_fast_path.py::test_fast_path_guards_mutations_but_not_reads`
- `tests/main/test_bead_fast_path.py::test_fast_path_refuses_unsafe_resolved_location_before_rust`
- `tests/main/test_bead_fast_path.py::test_fast_path_refuses_mutation_from_plain_checkout_sidecar_record`

  These three assert that the write-sandbox guard raises before the Rust binding is reached. Switch each to a mutating
  verb that still fast-paths — `rm` is the closest analogue, and `["rm", "beads-1"]` needs no plan file. Update the
  `match=`/`assert` strings from `"fast-path create"` to the chosen verb.

- `tests/main/test_bead_fast_path_mutations.py::test_fast_path_create_and_rm_use_rust_on_sidecar_layout`

  This one really does exercise create-through-Rust. Seed its two beads with `BeadProject.create` (the module already
  imports `BeadProject` and `storage_plan_path`) instead of `try_handle_bead_fast_path(["create", ...])`, keep the `rm`
  half and its `summaries[-1]` assertions intact, and rename the test to match what it now covers. Preserve the existing
  `design == storage_plan_path(plan.resolve())` assertion by passing the same design path to `BeadProject.create`.

`tests/main/test_bead_fast_path_context.py` line ~181 calls `_resolve_fast_path_context(["create", ...])` directly and
is unaffected; leave it alone.

**Add the missing dispatch regression test.** The gap that let this ship is that no test drove `sase bead create`
through `main()`. Add to `tests/main/test_bead_fast_path.py` (or a sibling module, whichever reads better next to the
existing dispatch coverage):

- `try_handle_bead_fast_path(["create", "--title", "…", "--type", "task"])` returns `None`, mirroring the shape of the
  existing `test_fast_path_defers_list_to_argparse`, with a `_resolve_fast_path_context` stub that fails if called so
  the decline is proven to happen before any store resolution.
- An end-to-end test that runs the real CLI dispatch with `SASE_AGENT_NAME` set and asserts the stored bead's
  `created_by` is the globalized agent name. Drive it through `sase.main.entry.main` with a patched `sys.argv` inside
  the existing bead-store test fixtures (catching the `SystemExit` that `main` raises), so a future fast-path change
  that re-swallows `create` fails here. Do not settle for calling `handle_bead_create` directly — that is precisely the
  test shape that missed the defect.

**Validation for this phase:** `just install`, then
`.venv/bin/python -m pytest tests/main/test_bead_fast_path.py tests/main/test_bead_fast_path_mutations.py tests/main/test_bead_fast_path_context.py tests/test_bead -q`.

## Phase 2 — Remove the superseded `SASE_AGENT` guard

**`src/sase/bead/attribution.py`**:

- Delete the `_UNTRUSTED_IDENTITY_SOURCES` constant and its explanatory comment.
- Reduce `acting_agent_name` to: discover the identity, return `None` when there is none, `validate_new_agent_name`, and
  return `globalize_owned_agent_name(name)` — all still inside the `try` that returns `None` on any failure.
- Drop `agent_name_from_meta` from the `sase.agent.identity` import; keep `discover_agent_identity`.
- Replace the removed comment with a one-line note recording that `discover_agent_identity` already ignores the bare
  `SASE_AGENT` launcher marker (bead `sase-bp`, commit `14d66229a`), so this resolver does not re-filter identity
  sources.

Leave `tests/test_bead/test_attribution.py` unchanged. Confirm all four `SASE_AGENT`-related tests still pass —
`test_acting_agent_name_ignores_bare_sase_agent_flag` and
`test_acting_agent_name_prefers_meta_over_bare_sase_agent_flag` are the ones that matter. If either fails, stop: that
means `discover_agent_identity` does not in fact subsume the guard, and the branch must be restored instead.

**Validation for this phase:** `.venv/bin/python -m pytest tests/test_bead/test_attribution.py -q` and
`just _lint-symvision` (the import removal must not leave an unused symbol).

## Phase 3 — Land epic sase-bv

Run only after phases 1 and 2 are committed.

**File the follow-up task beads** carried over from the epic's `PROPOSED FOLLOW-UP:` notes that are still open. The
other seven collected entries were verified already resolved and must not be re-filed; the close note records that.

1. ```bash
   sase bead create -T task -t 'Refresh 6 stale ACE PNG snapshot goldens' -d '<details>'
   sase bead update <id> -s ready
   ```

   `just test-visual` fails 6 of 393 cases on a clean master:
   `test_ace_png_snapshots_agents_panel_clan_collapse.py::test_selected_panel_clan_collapse_precedes_status_group_png_snapshot`,
   `test_ace_png_snapshots_agents_panels.py::test_agents_sole_selected_panel_png_snapshot`,
   `test_ace_png_snapshots_agents_panels.py::test_agents_collapsed_panel_png_snapshot`,
   `test_ace_png_snapshots_agents_slow_tools.py::test_agents_slow_tool_calls_fold_levels_png_snapshots`,
   `test_ace_png_snapshots_agents_tribe_panel.py::test_tribe_panel_display_config_png_snapshot`, and
   `test_ace_png_snapshots_agents_tribe_panel.py::test_tribe_panel_four_level_png_snapshots`. Proposed by `sase-bv.2`
   and `sase-bv.4`, which reported 53 failures; the tribe-panel commits `574b7761f` and `7e9527c7a` have since fixed 47
   of them, leaving this residue. Note in the bead that the fix is to confirm the remaining diffs are the intended
   tribe-header layout change and then accept them with `--sase-update-visual-snapshots`.

2. ```bash
   sase bead create -T task -t 'Stabilize suite-gate SIGKILL capacity test under full-suite load' -d '<details>'
   sase bead update <id> -s ready
   ```

   `tests/test_suite_gate_integration.py::test_scaled_suite_runs_share_capacity_and_release_after_sigkill` timed out
   during a full `just test` run for `sase-bv.4` after its child pytest had already reached 100%. It passes in isolation
   in ~4 s, so this is contention under parallel load, not a logic failure. Proposed by `sase-bv.4`. Note the sibling
   in-progress flake bead `sase-bt` in the description.

**Close the epic**, recording what was verified and what was fixed:

```bash
sase bead close sase-bv --note "<note>"
```

The note must state: all five phases were read against the source and confirmed landed; `sase bead create` did not
actually record the acting agent because the Rust bead-CLI fast path bypassed `handle_bead_create`, and that was fixed
and covered by an end-to-end dispatch test; the superseded `SASE_AGENT` guard in `attribution.py` was removed after bead
`sase-bp` fixed the hazard at its source; and that seven of the nine collected follow-ups (stale `sase-bj.3` symvision
exemptions, generated provider-skill drift, the missing prompt-to-plan link, model-completion label casing, the
`prefix_policy` privatization, and both bead-page publication performance reports, the last of which was fixed by bead
`sase-c7` / commit `c82eff9a0`) were re-verified as already resolved and deliberately not re-filed.

**After the close succeeds**, run `just symvision` and remove any stale epic-symbol whitelist entries or now-unused code
it reports for `sase-bv`. The epic's entries were already removed by `sase-bv.3` and symvision is currently clean, so
expect no findings; if any appear, clean them.

**Finally**, set `status: done` in the frontmatter of the epic's plan file,
`plans:202607/bead_created_by_attribution.md` (the `PLAN` path printed by `sase bead show sase-bv`).

If the close is rejected, do not force it. The named phases were never completed; finish or reopen them first.

## Validation

Run `just install` before anything else — workspace directories are ephemeral and dependencies may have moved.

Run `just check` before finishing. Two pre-existing failures are expected and are the subject of the follow-up beads
filed in phase 3; nothing else may fail:

- the 6 ACE PNG snapshot goldens listed above (these surface in `just test-cov` / `just test-visual`, not `just check`),
- the suite-gate SIGKILL capacity test, only under full parallel load.

The work is done when, from a SASE agent, `sase bead create -T task -t "…"` stores that agent's global name in
`created_by`, `sase bead show <id>` renders it under `CREATED BY` with a working agents-sidecar link, `just check` is
green, epic `sase-bv` is closed, and its plan file reads `status: done`.
