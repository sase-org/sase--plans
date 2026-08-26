---
tier: tale
title: Add `<epic>..` expansion to `sase bead show`, and make a multi-bead show fast
goal:
  "`sase bead show tt..` renders the epic plus every one of its direct children, and a
  nine-bead show costs roughly one bead's work instead of nine."
size: medium
proposed_by: bbugyi200.athena.0e1.f1
---

# Plan: `<epic>..` expansion for `sase bead show`

## Goal

`sase bead show tt..` must be exactly equivalent to

```
sase bead show sase-tt sase-tt.1 sase-tt.2 sase-tt.3 sase-tt.4 sase-tt.5 sase-tt.6 sase-tt.7 sase-tt.8
```

and it must not be slow. Those are two separate pieces of work that belong in one
change, because the expansion turns "show many beads at once" from a thing users
occasionally type into the command's headline feature, and today that path is
quadratically wasteful for a reason that has nothing to do with the bead store.

## Background

### What exists today

`sase bead show` already accepts multiple IDs. `register_bead_show_parser()`
(`src/sase/main/parser_bead_queries.py:340`) declares `ids` with `nargs="+"`;
`handle_bead_show()` (`src/sase/bead/cli_query.py:261`) hands them to
`resolve_show_batch()` / `render_show_batch()` (`src/sase/bead/cli_show_batch.py`),
which resolve in argv order, collapse duplicates by canonical ID, and report misses on
stderr with exit 1 without suppressing the beads that did resolve. Full IDs and
dash-free shorthand both work (`tt` resolves to `sase-tt`).

Epic children are already a first-class concept in this command. `IssueDetail` carries
`phases` and `child_epics` (`src/sase/bead/cli_detail_resolution.py:30`), the full
render prints them under `CHILDREN`, and the JSON envelope emits them as
`children.phases` / `children.epics`. `BeadProject.get_epic_children()`
(`src/sase/bead/_project_queries.py:174`) returns the direct children of one bead, and
`src/sase/bead/phase_selector.py` already resolves `<epic> --phases 1,3,5-7` for
`sase bead close`. So the data and the precedent for a phase-selector grammar both
exist; what is missing is a way to say "this epic and everything under it" in one argv
token.

`sase bead show` is deliberately **not** on the Rust fast path —
`try_handle_bead_fast_path()` returns `None` for `show`
(`src/sase/main/bead_fast_path.py:49`), so argparse and the Python handler own the
command end to end. The dormant Rust `handle_show` arm defers whenever
`args.len() != 1`, so it never sees a multi-ID invocation and is not part of this
change. That is the same boundary the already-landed multi-ID feature works within.

### Measured: a nine-bead show costs 12 seconds, and the bead store is not why

All numbers below were measured on this project's real bead store (4,335 beads), showing
`sase-tt` plus its eight phases.

| invocation                             | wall clock |
| -------------------------------------- | ---------- |
| `sase bead stats` (fast-path baseline) | 0.57s      |
| `sase bead show` — 1 bead              | 2.59s      |
| `sase bead show` — 2 beads             | 3.72s      |
| `sase bead show` — 3 beads             | 4.86s      |
| `sase bead show` — 5 beads             | 7.26s      |
| `sase bead show` — 9 beads             | 11.97s     |

That is a clean straight line at **~1.17s per additional bead**, and `--no-links` barely
moves it (10.88s for nine). A `cProfile` run of `handle_bead_show()` over the same nine
beads attributes the time as:

| call site                                                             | calls | cumulative |
| --------------------------------------------------------------------- | ----- | ---------- |
| `resolve_bead_creator_url` (`src/sase/bead/cli_detail_context.py:58`) | 9     | **13.36s** |
| `bead_show_issue_detail` (the Rust store read)                        | 9     | 2.51s      |
| everything else                                                       | —     | ~1.4s      |

**77% of the command is creator-URL resolution, and it is not the bead store.**
`resolve_bead_creator_url` → `HostedLinkResolver.agent_url` → `sase_agent_ref_for_name`
→ `_is_reserved_family_name` (`src/sase/sase_agent.py:136`) →
`get_reserved_family_names()` → `_load_registry_for_reservations()`
(`src/sase/agent/names/_registry.py:501`). That loader is the **name-allocation** path:
it calls `_reset_registry_scan_caches()` and passes `trust_stale_proof_memo=False`, so
every call re-proves registry freshness by walking every project's artifact directories
— 726,095 `posix.stat` calls across the nine beads. It is correct to pay that when
_allocating_ an agent name. `sase bead show` is only asking "is this creator's name a
family container, so I can spell its sidecar URL", which is a display read.

The fix already exists and is already used elsewhere. `name_registry_load_session()`
(`src/sase/agent/names/_registry.py:429`) reuses one validated registry snapshot for a
bounded operation; `src/sase/sdd/associations/_build.py:53`,
`src/sase/agents_sync/git_sync.py:338`, and
`src/sase/agents_sync/commit_publication_transaction.py:72` all wrap their work in it.
`handle_bead_show` does not. Wrapping it, measured on the same store:

| invocation | today          | inside a load session |
| ---------- | -------------- | --------------------- |
| 1 bead     | 1.44s – 2.09s  | **0.41s**             |
| 9 beads    | 9.82s – 10.09s | **3.17s – 3.41s**     |

Output was byte-identical in both cases (45,202 characters). This is a ~3x win for the
nine-bead case and a ~4x win for the _single_-bead case that every agent and every human
already pays on every `sase bead show`.

After that fix the remaining nine-bead cost is dominated by the nine separate Rust
`bead_show_issue_detail` calls (2.5s), each of which reduces the whole event store. A
batched core read would be the next lever; see [Non-goals](#non-goals).

## Design

### D1. Grammar: one trailing `..`, nothing else

An argv token that ends in `..` is an expansion token. The stem is everything before the
final two dots, and it accepts exactly what a plain ID argument accepts today — full
(`sase-tt..`) or dash-free shorthand (`tt..`).

The stem must be non-empty and must not itself end in `.`. That single rule rejects the
two malformed spellings a user can plausibly type:

| token          | stem         | outcome                                                           |
| -------------- | ------------ | ----------------------------------------------------------------- |
| `tt..`         | `tt`         | expand                                                            |
| `sase-tt..`    | `sase-tt`    | expand                                                            |
| `sase-tj.10..` | `sase-tj.10` | expand (a child epic is a valid target)                           |
| `..`           | `""`         | invalid-expansion error                                           |
| `tt...`        | `tt.`        | invalid-expansion error                                           |
| `..tt`         | —            | not an expansion token; an ordinary ID, resolves and fails as one |
| `tt`           | —            | not an expansion token; unchanged behavior                        |

`..` is not a shell glob metacharacter, so `sase bead show tt..` needs no quoting in
bash or zsh.

### D2. What expands: the bead plus its **direct** children

`<id>..` renders the target bead followed by every direct child — phase beads _and_
child epic plan beads — in one level only.

The user's request said "child phase beads", and for the worked example the two are the
same thing: `sase-tt` has eight phase children and no child epics, so `tt..` produces
exactly the nine IDs asked for. They diverge for an epic like `sase-tj`, which has nine
phases _and_ a child epic `sase-tj.10`. Including all direct children is the better rule
there: the epic's own `CHILDREN` section already lists `sase-tj.10`, so a `..` that
silently skipped it would be lossy in a way the user could not see from the output. If
that turns out to be unwanted, restricting to `IssueType.PHASE` is a one-line change.

Expansion is **not** recursive. `sase-tj..` yields `sase-tj`, `sase-tj.1` …
`sase-tj.10`, but not `sase-tj.10.3`. `get_epic_children()` is already a `parent_id`
lookup, so direct-only falls out of the existing query; recursion would need a new
traversal and would make output size unbounded and unguessable. A user who wants the
grandchildren writes `sase-tj.. sase-tj.10..`.

### D3. Child order: numeric phase order, not creation order

`get_epic_children()` orders by `created_at`, which usually but not always matches phase
order, and which sorts `.10` correctly only by luck. Sort children by the integer suffix
after the final dot when it parses as digits, and keep everything else after those in
the order the store returned. That puts `sase-tj.10` after `sase-tj.9` rather than after
`sase-tj.1`, which lexical sorting would get wrong.

### D4. Expansion belongs inside `resolve_show_batch`

Put it in `resolve_show_batch()` rather than in `handle_bead_show()`. That function
already receives the `view`, already owns the `_ShowFailure` channel, already dedupes by
canonical ID, and already computes `multi_requested` — every mechanism expansion needs.
Doing it there means `handle_bead_show()` gains no new failure plumbing.

Concretely, per argv token: a non-expansion token contributes itself, unchanged. An
expansion token contributes `[stem, *ordered_child_ids]`. Note the **stem** is passed
through rather than a canonicalized ID: `view.get_epic_children()` already canonicalizes
internally, so passing the stem avoids a second store read and avoids requiring
`resolve_id` on the `view` protocol, which the existing test doubles do not implement.
Downstream dedup is by canonical ID, so a shorthand stem and a full child ID still
collapse correctly against anything else in the batch.

### D5. Errors reuse the existing channel

- Stem does not resolve. This splits by ID form, and both halves converge on the same
  message — verified against the real store rather than assumed:
  - A **shorthand** stem (`nope..`) makes `BeadProject.get_epic_children()` raise
    `KeyError('Issue not found: nope')` from its internal `resolve_id`. Catch it and
    record `issue not found: <stem>`.
  - A **full-form** stem (`sase-nope..`) raises nothing: `resolve_id` returns an
    unrecognized full ID unchanged and the children query comes back empty. Expansion
    therefore yields `[stem]`, and the existing resolve loop fails it with exactly
    `Error: issue not found: sase-nope`. No new code is needed for this half.

  Either way the reported ID is the **stem**, not the raw token, because the stem is the
  bead that was not found; and stderr, exit 1, and "beads that did resolve still print"
  are all unchanged.

- Malformed token (D1): a failure reading
  `invalid ID expansion: '<token>' (expected <epic-id>.., for example sase-tt..)`.
- Target resolves but has no children: **not** an error. Render the bead alone. An epic
  with no phases authored yet is a legitimate target, and `sase bead close --phases`
  erroring on a non-epic is a different situation — that flag mutates, this one
  displays.

### D6. `--format json` always emits an array for an expansion

`resolve_show_batch` sets `multi_requested = len(ids) > 1` today, which drives the JSON
envelope-vs-array choice. After expansion that must become
`len(expanded_ids) > 1 or any_token_expanded`.

The second clause matters: without it, `sase bead show X.. --format json` would emit an
array for an epic with phases and a bare object for an epic without them, so a script
could not know which shape it was parsing without inspecting the store first. A `..`
token is a request for a _set_, so it always yields the array form.

### D7. Ordering and duplicates are positional, and already specified

Each token expands in place, and the existing first-occurrence-wins dedup applies to the
expanded list. So `sase bead show tt.3 tt..` renders `sase-tt.3`, then `sase-tt`, then
`sase-tt.1`, `sase-tt.2`, `sase-tt.4` … — `sase-tt.3` is not repeated. This needs no new
code, only documentation.

### D8. The load-session fix

Wrap the body of `handle_bead_show()` in `name_registry_load_session()`. It must enclose
`render_show_batch()`, not just resolution, because creator URLs are resolved during
rendering (`creator_url_for` is called per entry inside `_render_full_batch`).

Import it from `sase.agent.names._registry`, matching all three existing call sites. If
symvision objects to the private-module import, the tidier resolution is to re-export
`name_registry_load_session` from `sase.agent.names` (its `__init__` already re-exports
about twenty other `_registry` names, but not this one) — read
`sase/memory/symvision.md` with `/sase_memory_read` before deciding.

This is safe for this command specifically: `sase bead show` is a read-only, short-lived
process, and the session's entire purpose is to reuse one validated snapshot for a
bounded operation. It does not touch the allocation path, which keeps calling
`_load_registry_for_reservations()` and keeps paying the full proof.

## Implementation steps

1. **New module `src/sase/bead/show_epic_expansion.py`.** Export the `..` suffix
   constant, a `expansion_stem(token) -> str | None` parser implementing D1, an
   `ExpansionError` for the malformed case, and
   `expand_epic_target(view, stem) -> list[str]` implementing D2/D3 on top of
   `view.get_epic_children()`. Keep it Textual-free and store-agnostic — it only needs
   `get_epic_children` off the view.

2. **`src/sase/bead/cli_show_batch.py`.** In `resolve_show_batch()`, expand each argv
   token before the existing resolve loop (D4), route `ExpansionError` and the shorthand
   `KeyError` into `_ShowFailure` (D5), and compute `multi_requested` from the expanded
   list plus an `expanded_any` flag (D6).

3. **`src/sase/bead/cli_query.py`.** Wrap `handle_bead_show()` in
   `name_registry_load_session()` (D8). No other change to this function.

4. **`src/sase/main/parser_bead_queries.py`.** Document `<epic-id>..` in the `show`
   description and add `sase bead show sase-tt..` to the epilog examples, keeping the
   existing example ordering. `sase/memory/cli_rules.md` requires the `-h` output to
   stay clear and complete; no new option is added, so its short-alias rule does not
   apply.

5. **`src/sase/bead/cli_admin.py`.** Add one row to the `sase bead` command listing
   around line 412, next to the existing `sase bead show <id>` rows.

6. **`docs/beads.md`.** Extend the `sase bead show <id> [<id2> ...]` section (line 1592)
   with the expansion grammar, the direct-children rule, child ordering, the JSON array
   rule, and the positional-dedup example from D7. **Do not change that heading** — it
   is the anchor `#sase-bead-show-id-id2`, linked from lines 1196 and 1513 of the same
   file.

## Testing

- **New `tests/test_bead/test_cli_show_epic_expansion.py`**, following the `_View`
  test-double pattern already in `tests/test_bead/test_cli_show_multi.py` (whose double
  already implements `get_epic_children`, so it extends cleanly). Cover: expansion of an
  epic with children; shorthand stem; an epic with no children rendering alone; `..` and
  `tt...` producing the invalid-expansion failure and exit 1; a nonexistent stem
  producing `issue not found: <stem>` and exit 1 while other beads still render —
  **both** the shorthand form that raises and the full-ID form that does not (D5); `.10`
  ordering after `.9`; a child epic included among the children; and the
  positional-dedup case from D7.
- **JSON shape**: assert `X.. --format json` emits an array even when the epic has
  exactly one child and when it has none (D6).
- **Golden CLI case** in `tests/test_bead/test_cli_golden.py`: the `current` fixture
  store already contains `beads-1` (epic) with `beads-1.1` and `beads-1.2` phases, so
  add `["bead", "show", "beads-1..", "--format", "compact"]` with a new
  `golden/cli/show_epic_expansion_compact.stdout`. Compact keeps the golden small and
  still pins order and membership.
- **Load-session regression**: assert the registry load session is active while
  `handle_bead_show` resolves creator URLs — monkeypatch `resolve_bead_creator_url` in
  `sase.bead.cli_query` and assert
  `sase.agent.names._registry._LOAD_SESSION_ACTIVE.get()` is `True` when it fires. (Only
  the `ContextVar` exists today; if a public predicate reads better, add one next to
  `name_registry_load_session`.) This is the test that stops the fix from being silently
  reverted, since nothing else observes it.

## Non-goals

- **Recursive expansion.** D2. A second syntax for "the whole subtree" can be added
  later if anyone asks; nothing here forecloses it.
- **`..` on other bead subcommands.** `close`, `update`, `rm`, `dep` are all mutating,
  and a mutation that silently widens to every child is a different and much riskier
  feature. `sase bead close` already has `--phases` for the explicit version.
- **A batched Rust detail read.** After D8 the nine-bead show spends ~2.5s in nine
  separate `bead_show_issue_detail` calls, each reducing the entire event store; a
  `bead_show_issue_details` batch binding in `sase-core` would collapse those to one.
  That is a real and worthwhile follow-up, but it is a cross-repo change with its own
  release floor, it helps every multi-ID show rather than this feature specifically, and
  D8 already removes the larger share of the cost. It should be filed as its own task
  rather than bundled here.
- **Shell completion for `..` tokens.** Bead-ID completion lives in
  `src/sase/completion/candidates/` and is untouched; `tt..` is typed, not completed.
- **Rewriting the epic-close xprompt.** `src/sase/default_config.yml` (around lines
  1356-1357) tells the closing agent to "run `sase bead show` on every child", which
  this syntax collapses into one command. That is an obvious follow-up, but xprompt text
  has its own golden coverage and it is not needed for the feature to work.

## Risks

- **Golden churn.** Only the one new compact golden is added; existing `show*.stdout`
  goldens are untouched because no existing invocation changes shape. Adding `..` to the
  parser description does change `sase bead show --help`;
  `tests/main/test_parser_command_help.py` does not assert on the show description
  today, but confirm that before landing.
- **The load session hiding a genuinely stale registry.** Bounded to one short-lived
  read-only process, and byte-identical output was verified on a nine-bead render. The
  allocation path is untouched.
- **`--pager always` and long output.** `tt..` on a large epic produces a lot of text,
  so it is exactly the case the pager exists for. The pager's `SASE_AGENT` guard
  (`src/sase/cli_pager.py:115`) already disables paging under an agent in `auto` mode,
  so agents get the expanded output straight to stdout. No change needed.

## Verification

- `just check` after implementation. Run `just install` first if the workspace has not
  been used recently.
- `just check-full` through `/sase_monitor` before landing — this touches the shared
  `sase.agent.names` registry loader, whose callers reach well beyond the bead CLI.
- Record the before/after timing for a real nine-bead expansion in the implementing
  agent's completion note, so the perf claim in this plan is confirmed rather than
  assumed.
