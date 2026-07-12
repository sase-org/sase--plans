---
create_time: 2026-07-12 16:29:55
status: done
prompt: 202607/prompts/plan_list_status_and_limit.md
tier: tale
---
# `sase plan list`: Status Filtering and Per-Status Limits

## Problem

`sase plan list` renders a fixed dashboard: every pending proposal, plus the 10 most recent approved plans and the 10
most recent inferred-rejected plans (both hardcoded via `_APPROVED_LIMIT` / `_REJECTED_LIMIT` in
`src/sase/main/plan_inventory.py`). Users cannot:

1. **See more history** — there is no way to ask for the last 25 (or all) approved/rejected plans.
2. **Focus on one status** — there is no way to show only the Approved section (or only Rejected, or only the actionable
   Proposed queue) without visually wading through the full dashboard.

## Goals

- Intuitive: flags that read naturally, mirror existing `sase plan search` flags, and compose with the existing
  `-t/--tier` filter.
- Reliable: no silently truncated results, no degraded rejected-plan inference when filters are active, stable JSON.
- Beautiful: the filtered dashboard should still look like a deliberate dashboard, not a stripped-down afterthought.

## CLI Surface (Design)

Two new options on `sase plan list` (and therefore on bare `sase plan`, which delegates to `list`):

```
sase plan list [-j] [-n LIMIT] [-s STATUS ...] [-t TIER ...]
```

| Option                                     | Semantics                                                                                                        |
| ------------------------------------------ | ---------------------------------------------------------------------------------------------------------------- |
| `-s/--status {approved,proposed,rejected}` | Repeatable section filter. Only the selected sections are rendered/serialized. Default: all three.               |
| `-n/--limit N`                             | Max rows per _history_ section (Approved and Rejected, each). `0` = unlimited. Default: `10` (current behavior). |

Design rationale:

- **Flag consistency**: `sase plan search` already uses `-s/--status` (repeatable) and `-n/--limit` (nonnegative, `0` =
  unlimited, via the shared `nonnegative_int` type). Reusing the same letters and semantics keeps the `plan` command
  group coherent. The status _vocabulary_ differs per subcommand on purpose: `list` filters pipeline sections
  (`proposed|approved|rejected`), while `search` filters plan-file frontmatter (`wip|done`) — each help text says which
  it means.
- **Repeatable, not comma-separated**: matches the existing `--tier` append pattern (`-s approved -s rejected`).
- **`--limit` never hides Proposed rows**: pending proposals are the actionable set — their `id_prefix` values feed
  `sase plan approve/reject`, and hiding one silently would be a footgun. Proposed always shows every pending proposal
  (the queue is naturally small). The help text states this explicitly: "Maximum approved and rejected rows shown per
  section; 0 = unlimited; pending proposals are always shown in full (default: 10)".
- Options stay alphabetically sorted in the parser/help (`--json`, `--limit`, `--status`, `--tier`), every long option
  keeps a short alias, and the epilog gains examples:
  - `sase plan list --status approved --limit 25`
  - `sase plan list -s proposed`
  - `sase plan list -s rejected -n 0`

### Considered and rejected

- `--all` as sugar for `--limit 0`: redundant with `-n 0`; keeps help tighter.
- `pending` as an alias for `proposed`: strict `choices` keep help output and error messages self-explanatory.
- Per-status limits (`--approved-limit`, `--rejected-limit`): more surface for little benefit; one shared per-section
  limit composes with `--status` to express everything (`-s approved -n 50`).

## Behavior Semantics

### Status filter is a _view_, not a _collection_ change

The inventory always collects all three sections internally, regardless of `--status`:

- Rejected inference requires the proposed + approved plan keys (`represented_paths`) to decide which archived proposals
  are "not represented" — so `--status rejected` still needs the approved collection to be correct.
- The summary strip keeps consistent counts whether or not a filter is active (no "counts changed because I filtered"
  surprises).
- Proposed collection is cheap (pending notification store) and the approved scan is already the default cost of every
  `sase plan list` today, so there is no meaningful perf win to chase by skipping collection.

`--status` only controls which sections are rendered (Rich) or serialized (JSON).

### Limit semantics and the approved scan cap (no silent truncation)

`build_plan_inventory()` gains a single `limit` input that feeds both `approved_limit` and `rejected_limit` (`0` →
unlimited). The approved collector currently stops scanning after `_APPROVED_META_CANDIDATE_LIMIT = 2_000` newest-first
artifact meta files — fine for 10 rows, but a correctness hazard for large/unlimited requests:

- **Finite limit**: scale the candidate cap to `max(2_000, 100 * limit)` so realistic requests (`-n 50`, `-n 200`)
  always have headroom.
- **`--limit 0` (unlimited)**: scan the full artifact history — the user explicitly opted into an exhaustive listing, so
  exhaustive-and-slower beats fast-and-wrong.
- **Disclosure**: if a finite-limit scan exhausts the candidate cap before finding `limit` distinct approved plans, the
  inventory records it (e.g. `approved_scan_truncated`) and the dashboard prints a dim note under the Approved panel
  ("scanned the newest N agent artifacts; older approvals may exist"); JSON exposes the same flag in `summary`. Silent
  caps read as "covered everything" when they didn't.

Raising the approved limit also _improves_ rejected inference (more approved plans become "represented", so fewer
archived files are misinferred as rejected) — a free reliability win worth a code comment where `represented_paths` is
built.

## Rendering (the "beautiful" part)

Building on the existing Rich dashboard (`render_plan_inventory`):

1. **Summary strip stays put**: all four counters (Proposed / Approved shown / Rejected shown / Archived total) always
   render so the strip's geometry is stable; when a status filter is active, the _unselected_ counters render dim so the
   eye lands on what was asked for.
2. **Only selected panels render** when `--status` is active — no husk panels with "No … found" noise for sections the
   user excluded.
3. **Panel titles gain counts**: `Proposed (2)`, `Approved (25)`, `Rejected (3)` — scannable at a glance, and honest
   about how many rows follow.
4. **One consolidated `Filters:` line** replaces the current `Tier filter:` line, shown only when any non-default
   setting is active, e.g. `Filters: status=approved,rejected · tier=epic · limit=25` (tier keeps its existing per-tier
   colored counts). One place to look to understand "why does the output look like this".
5. Section colors stay as-is (yellow/green/red) — they already encode status well.

## JSON Projection

- With `--status`, only the selected section arrays are emitted (omitted ≠ empty: `jq .approved` returning `null`
  clearly signals "not requested", where `[]` would falsely read as "none exist").
- `summary` always keeps the four existing counts; it additionally includes `"status_filter": [...]` when a status
  filter is active and `"limit": N` when the limit differs from the default — mirroring the existing conditional
  `tier_filter` pattern — plus `"approved_scan_truncated": true` when the disclosure case fires.
- No changes to row shapes, so existing consumers of the default invocation are unaffected.

## Implementation Outline

All changes live in this repo's Python CLI layer — `plan_inventory.py` is existing local presentation/inventory logic
(pending-notification store + artifact metas), not shared cross-frontend domain behavior, so no Rust core changes are
needed.

1. **`src/sase/main/parser_plan.py`** — add `-n/--limit` (`nonnegative_int`, default 10) and `-s/--status`
   (`action="append"`, `choices=("approved", "proposed", "rejected")`) to the `list` subparser; keep options
   alphabetical; extend the `list` description + epilog examples.
2. **`src/sase/main/plan_list_handler.py`** — thread `args.limit` and `args.status` into `build_plan_inventory()`.
3. **`src/sase/main/plan_inventory.py`** —
   - `build_plan_inventory(*, limit, tiers, statuses)`: normalize/dedupe statuses, map `limit` to both section limits,
     record `status_filter`, `limit`, and `approved_scan_truncated` on `_PlanInventory`.
   - `_collect_approved_plans`: scaled/unbounded candidate cap + truncation detection as designed above.
   - `plan_inventory_to_json` / `render_plan_inventory`: section selection, dim summary cells, panel-title counts,
     consolidated `Filters:` line, truncation note.
4. **Docs** — update the `sase plan` flag table + prose in `docs/configuration.md` (new flags, JSON key behavior under
   filtering) and the one-line description in `docs/cli.md` if needed.

## Testing

- **Parser** (`tests/main/test_parser_plan.py`): new options parse (repeatable status, limit default/`0`, invalid
  status/negative limit rejected); help-completeness assertions extended for the new flags/examples; the existing
  short-alias sweep covers the new options automatically.
- **Inventory** (`tests/test_plan_inventory.py`):
  - `limit` overrides both section limits; `0` yields all approved/rejected rows; proposed is never limited.
  - Status filter: JSON omits unselected sections, `summary.status_filter` present, counts unchanged; rejected-only
    filtering still produces correct inference (approved still collected internally).
  - Scan-cap scaling + `approved_scan_truncated` disclosure (finite limit hitting the cap; unlimited scanning past 2,000
    candidates).
  - Composition with `--tier` (both filters active).
- **Rendering**: extend the existing stable-columns/empty-state render tests — filtered run renders only selected
  panels, dim summary cells, panel-title counts, `Filters:` line replaces `Tier filter:` line (update the tier-filter
  render assertion accordingly).
- Run `just check` before finishing.
