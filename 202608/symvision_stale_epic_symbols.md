---
tier: tale
title: Clear the stale symvision epic-symbol whitelist entries
goal:
  just symvision passes on master again because the Justfile's --epic-symbol whitelist
  carries only entries whose beads are still open and whose symbols still lack a real
  non-test consumer.
size: small
proposed_by: bbugyi200.athena.0ck
---

- **AGENTS:**
  - [bbugyi200.athena.0ck](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0ck.md)

# Clear the stale symvision `--epic-symbol` entries that break `just symvision`

## Problem

`just symvision` fails on a clean `master` tree, which also reddens the `symvision`
stage of `just lint` and `just check` for every agent in the repo, regardless of what
they changed.

The `_lint-symvision` recipe in `Justfile` passes a whitelist of
`--epic-symbol "<bead_id>(<symbol>)"` exemptions. Those exemptions are _self-cleaning_:
symvision fails the run when an entry's bead is missing or closed, when the symbol is
now properly used by a non-test consumer, or when the symbol no longer exists as a
public definition. Four of the six entries have gone stale as their epic phases landed
and closed, so symvision now errors instead of linting.

## Root cause

Whitelist entry lifetime is tied to bead status plus real usage, and nothing on the
landing path removes an entry when the phase that needed it lands:

- `sase-sp.3(FinalizerDeferralWire)` and `sase-sp.3(finalizer_deferral_from_dict)` —
  bead `sase-sp.3` closed `2026-08-24T16:09:32Z`, and its landing commit `524d8f26f`
  wired both symbols into `src/sase/finalizers/declaration_manifest.py`. That commit did
  not touch `Justfile`.
- `sase-su.2(plan_provider_drain)` and `sase-su.2(execute_provider_drain)` — bead
  `sase-su.2` closed `2026-08-24T16:18:36Z`. Both symbols already had a real non-test
  consumer from phase 1's landing commit `bf3206b8f` (`src/sase/agent/provider_drain.py`
  re-exports them from `_drain_planning` / `_drain_execute`), so these two entries were
  unnecessary from the moment they were added. That is why the failure first surfaced as
  `symbol 'plan_provider_drain' is already properly used` on a tree where `sase-su.2`
  was still open, and has since escalated to the closed-bead error.

The `sase-su.2` phase agent's own workspace did remove its two lines (see its closing
note on the bead), but that tree is not on `master`, so `master` stayed red.

## Evidence gathered

Reproduced in workspace at `master` `524d8f26f` after `just install`:

```
Error: --epic-symbol 'sase-sp.3(FinalizerDeferralWire)': bead 'sase-sp.3' is closed. Remove this stale --epic-symbol entry and clean up the symbol.
Error: --epic-symbol 'sase-sp.3(finalizer_deferral_from_dict)': bead 'sase-sp.3' is closed. Remove this stale --epic-symbol entry and clean up the symbol.
Error: --epic-symbol 'sase-su.2(plan_provider_drain)': bead 'sase-su.2' is closed. Remove this stale --epic-symbol entry and clean up the symbol.
Error: --epic-symbol 'sase-su.2(execute_provider_drain)': bead 'sase-su.2' is closed. Remove this stale --epic-symbol entry and clean up the symbol.
```

Rerunning the exact same symvision invocation with only the two surviving entries
(`sase-n4.5(ProviderDisableWriteOutcome)`, `sase-n4(get_usage_limit_config)`) prints:

```
All public/private classes/functions are used properly!
```

So deleting the four stale lines is the complete fix: no Python symbol is dead, and no
symbol cleanup is required despite what symvision's closed-bead message suggests.

Non-test consumers confirmed by grep:

- `src/sase/finalizers/declaration_manifest.py:18,20,46,406` uses
  `FinalizerDeferralWire` and `finalizer_deferral_from_dict`.
- `src/sase/agent/provider_drain.py:31-32,41-42` uses `plan_provider_drain` and
  `execute_provider_drain`.

Both remaining entries must be kept: `sase-n4` and `sase-n4.5` are still `IN_PROGRESS`
and symvision accepts both entries today.

## Plan

1. **Set up the workspace.** Run `just install` first — ephemeral `sase_<N>` workspaces
   drift and `symvision`, `sase`, and the `sase_core_rs` extension must all be current
   or the bead-status lookups in `tools/sase_bead` fail with an unrelated `ImportError`.

2. **Re-derive the stale set instead of trusting this plan's snapshot.** Other agents
   land concurrently and each landing can add or retire entries, so start from live
   state:

   ```bash
   sase bead epic-symbols --format json
   just _lint-symvision        # the authoritative list of stale entries
   ```

   For every entry symvision complains about, confirm the reason with
   `sase bead show <bead_id>` (closed vs. open) before editing.

3. **Delete only the stale lines from the `_lint-symvision` recipe in `Justfile`.** As
   of this plan that is exactly these four lines:

   ```
   --epic-symbol "sase-sp.3(FinalizerDeferralWire)"
   --epic-symbol "sase-sp.3(finalizer_deferral_from_dict)"
   --epic-symbol "sase-su.2(plan_provider_drain)"
   --epic-symbol "sase-su.2(execute_provider_drain)"
   ```

   Keep the `sase-n4.5(ProviderDisableWriteOutcome)` and
   `sase-n4(get_usage_limit_config)` entries, keep both `--exclude-decorator` flags, and
   keep the trailing `{{ args }}` line and the line-continuation backslashes
   well-formed. If step 2 shows a line is already gone (the `sase-su.2` stitch may land
   first), that is the expected outcome — do not re-add it.

4. **Do not delete or privatize any Python symbol.** Symvision's closed-bead message
   ends with "and clean up the symbol", but here every affected symbol has a live
   non-test consumer (see Evidence). Before removing any symbol, `grep` for it under
   `src/` and stop if a non-test consumer exists. Deleting these symbols would break
   `src/sase/finalizers/declaration_manifest.py` and `src/sase/agent/provider_drain.py`.

5. **Do not touch `symvision` itself.** Per `sase/memory/symvision.md`, never patch the
   installed package, vendor a replacement, or change the dependency pin; fix the repo's
   own code.

6. **Verify.**
   - `just symvision` must print
     `All public/private classes/functions are used properly!` and exit `0`.
   - Then run `just check` per the repo's two-speed verification rule. `just check` is
     lint-heavy plus a diff-scoped test lane; hand it to `/sase_monitor` with
     `-s TESTING -S TESTED` and a `--next` action if it runs long. The `Justfile` edit
     is in the broadening set, so the scoped lane is expected to escalate to the full
     suite; if it does, run it through `/sase_monitor` as `just check-full`.

7. **Expect these unrelated failures; do not fix them here.** Both are already recorded
   as `PROPOSED FOLLOW-UP` notes on `sase-sp.3` / `sase-su.2`:
   - SASE validation `init memory --check` reports home memory / provider-shim drift and
     an unreferenced `sase/memory/obsidian.md`.
   - `tests/test_core_finalizer_facade.py::test_finalizer_facade_round_trips_deferred_instance_result`
     fails on the existing linked-core vs. Python finalizer deferred-result schema skew.

   Report them in the final summary; only investigate a failure that this `Justfile`
   edit could plausibly have caused.

8. **File one follow-up task bead for the systemic gap** via `/sase_new_task` (it
   de-dupes first; skip filing if it finds an existing match). The gap: the close-time
   guard `raise_if_leftover_epic_symbols` (`src/sase/bead/epic_symbols.py`, called from
   `src/sase/bead/cli_crud_lifecycle.py` and
   `src/sase/ace/tui/actions/_artifacts_beads_mutations.py`) parses the _closer's
   current working tree_ `Justfile`, discovered by walking up from `cwd`. That misses
   two real cases, both of which happened today: closing from a directory with no
   `Justfile` above it (`discover_justfile` returns `None`, so the guard silently passes
   — `sase-sp.3` closed with its entries intact on `master`), and closing from a
   workspace whose entry removal has not landed yet (`sase-su.2` cleaned its own tree,
   `master` stayed red). Suggested framing: the invariant should be checked against the
   landed tree or enforced on the landing path, not only against the closer's `cwd`. Use
   `-T "task(bug)"` with `location`, `repro`, and `impact` fields, and default the size
   per `sase_sizes.md` unless the root cause is certain.

## Risks

- **Merge conflict with the un-landed `sase-su.2` stitch.** That tree deletes its own
  two lines, which are adjacent to the `sase-sp.3` lines this change deletes. If the
  stitch lands after this change, expect a small conflict in the `_lint-symvision`
  recipe; resolve it by keeping _both_ deletions and preserving the `sase-n4` /
  `sase-n4.5` entries.
- **Over-deletion.** The most likely way to get this wrong is to follow symvision's
  "clean up the symbol" advice literally and delete live code. Step 4 is the guard.
- **Under-deletion.** Removing only the `sase-su.2` pair (the failure quoted in the
  original report) leaves `just symvision` red, because the `sase-sp.3` pair went stale
  later. Step 2's live re-derivation is the guard.

## Non-goals

- Changing symvision's behavior, its version pin, or the `--epic-symbol` mechanism.
- Fixing the bead close guard in this change (it gets a task bead in step 8).
- Fixing the pre-existing `init memory --check` drift or the finalizer deferred-result
  schema skew.
- Touching the `sase-n4` / `sase-n4.5` whitelist entries or the symbols they cover.

## Done when

- `just symvision` exits `0` on the working tree with the four stale entries gone.
- `just check` shows no symvision failure and no new failure attributable to this edit.
- The follow-up task bead for the close-guard gap exists (or `/sase_new_task` found an
  existing duplicate).
