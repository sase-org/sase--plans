---
tier: epic
title: Editor parity for fuzzy artifact-reference completion
goal: 'Every artifact-reference payload the ACE prompt input can find, an editor can
  find too: the same kinds, the same corpus, the same fuzzy queries, the same titles
  — and where a bound still applies, the response says so.

  '
phases:
- id: rank
  title: Fuzzy ranking in the agent and indexed-file collectors
  depends_on: []
  size: small
  description: 'rank: replace the case-insensitive starts_with prefilters in append_agent_page_candidates
    and append_artifact_index_candidates with the shared fuzzy matcher, ranking with
    compare_fuzzy and capping after ranking so prefix reach cannot regress.

    '
- id: titles
  title: Real titles on editor payload rows
  depends_on: []
  size: small
  description: 'titles: give each LSP payload row the title ACE shows (document frontmatter
    title, chat and artifact-file basename, bead title, agent short name) instead
    of echoing the payload, so title matching and the labelDetails/documentation affordance
    stop being dead code in editors.

    '
- id: reach
  title: Editor payload reach and disclosed truncation
  depends_on:
  - rank
  - titles
  size: medium
  description: 'reach: raise the per-root 200-row bound so editors see the corpus
    ACE sees, cache the inventory so the walk is not per-keystroke, and set truncated_payloads
    so a bound that remains is disclosed rather than silent.

    '
- id: land
  title: Docs, release, and epic landing
  depends_on:
  - reach
  size: small
  description: 'land: correct the docs/editor.md reachability claims, publish the
    sase-core release, run the editor acceptance checks, then close epic sase-b3,
    run just symvision, and mark both plan files done.'
parent_bead: sase-b3
create_time: 2026-07-30 06:56:58
status: done
bead_id: sase-b3.10
---

- **PROMPT:** [prompts/202607/editor_artifact_ref_parity.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/editor_artifact_ref_parity.md)
- **PARENT:** [202607/fuzzy_artifact_ref_completion.md](202607/fuzzy_artifact_ref_completion.md)
- **BEAD:** [sase-b3.10](https://github.com/sase-org/sase--beads/blob/main/pages/sase-b3/sase-b3.10.md)

# Plan: Editor Parity for Fuzzy Artifact-Reference Completion

## Problem

Epic `sase-b3` made artifact-reference completion fuzzy in the shared core menu and wired both the ACE prompt input and
the xprompt LSP onto it. Verified today, the ACE half is done: `@research:site` reaches
`202607/sase_sites_hub_and_pages/sase_sites_hub_and_pages.md` (180 matches, gold runs on `site`), `@rsch` resolves the
research kind, `@agent:sase-b3` returns 13 rows, `@file:panel` matches by file name, and empty-query order is unchanged.

The editor half is not. The LSP does not use ACE's catalog; `at_reference_payload_inventory` in
`crates/sase_xprompt_lsp/src/server.rs` builds its rows by calling `build_artifact_ref_payload_completion_candidates` in
`crates/sase_core/src/editor/completion.rs`, which was written for prefix completion and never got the epic's catalog
treatment. Three defects follow, all in that one function's collectors.

### 1. Two collectors still prefix-filter, so fuzzy rows never exist

The `lsp` phase dropped the `starts_with` prefilter from the document, chat, and bead collectors. It did not touch the
other two:

```rust
// append_agent_page_candidates
if name.is_empty()
    || !name.to_lowercase().starts_with(query)   // <-- still here
    || !seen.insert(name.clone())

// append_artifact_index_candidates
if !id.to_lowercase().starts_with(query) || !seen.insert(id.clone()) {   // <-- still here
```

Agent payloads are global names like `bbugyi200.athena.sase-b3.5`, so `@agent:sase-b3` prefix-matches nothing and the
inventory is empty before the fuzzy menu ever runs. Artifact-file payloads are `default:<hex>` / `explicit:<hex>` ids,
so `@file:` is reachable only by typing a digest prefix. ACE answers both queries.

The fix is not deletion. For documents, chats, and beads the prefilter was purely lossy, because
`bounded_relative_files` already caps its walk at `ARTIFACT_REF_MAX_RESULTS` files _before_ any matching. For agents and
indexed files the cap applies to _pushed candidates_ instead, so today a prefix query scans the whole directory and can
reach a row far past the first 200. Deleting these two prefilters would trade a fuzzy gap for a prefix regression.
Matching has to move into the collector: match, rank, then cap.

### 2. Every kind is bounded at 200 rows per root, silently

`ARTIFACT_REF_MAX_RESULTS = 200` bounds every collector, and for documents, chats, and beads it bounds the
_enumeration_, query-independently:

```rust
if files.len() >= ARTIFACT_REF_MAX_RESULTS {   // bounded_relative_files
    break;
}
```

So an editor sees at most 200 of the research sidecar's 305 markdown files and at most 200 of the `plans`
sidecar's 4336. This is the same defect the epic's Problem statement item 3 described for ACE — "Even the flat files are
half missing. Silently." — fixed there by per-role loading with a 5000-row cap and propagated truncation counts, and
left untouched here. `AtReferenceMenuWire.truncated_payloads` has no writer anywhere in `crates/sase_xprompt_lsp`, so
the count is always zero and nothing signals the bound.

### 3. Titles are always the payload, so the title affordance is dead code

`artifact_ref_candidate(name, insertion, …)` is called with the payload for **both** arguments by every collector, so
`AtReferencePayloadRowWire.label == payload` for every row. `build_payload_menu` then produces
`AtReferenceRowWire { label: payload, title: payload }`, and `at_reference_completion_item` correctly suppresses a title
that echoes its label:

```rust
let title = if row.title == row.label { "" } else { row.title.as_str() };
```

The condition is always true. `labelDetails.detail` is therefore never populated for an artifact reference, the
`documentation` preview never shows a title, and the menu's title matching — the half of `build_payload_menu` that lets
`@file:panel` find a screenshot by its file name — can never fire in an editor. ACE carries real titles for all five
kinds and matches them.

### Why this is `sase-b3`'s work

`docs/editor.md`, written by the epic's `land` phase, states without qualification:

> Matching is **fuzzy and ranked on the server**, so a payload is reachable by any memorable fragment of its path or
> title

for a paragraph that names document roles, chats, indexed artifact files, beads, and agents. That claim is false for
`@agent:` and `@file:` outright, false for "or title" on every kind, and misleading for any root over 200 files. Goal 1
of the epic covers both surfaces by name and goal 3 is not surface-qualified, and the epic's own Architecture section
states the constraint this violates: "the TUI and the LSP must rank identically." The epic shipped documentation it did
not deliver, so closing it is blocked on this plan.

## Goals

1. `@agent:` and `@file:` accept in an editor every query they accept in the ACE prompt input.
2. Editor payload rows carry the same titles ACE shows, and a title match surfaces the row.
3. A payload's reachability in an editor does not depend on where it sorts in a directory walk any more than it does in
   ACE; where a bound remains, the response discloses it.
4. No prefix regression. Every query that reaches a row today still reaches it.

## Non-Goals

- Changing the shared matcher, the menu, `AtReferenceRowWire`, or anything else in
  `crates/sase_core/src/editor/{fuzzy,at_reference}.rs`. Those are correct; only their inputs are wrong.
- `commit` and `bug`, which keep shape-validation-only treatment.
- The ACE prompt input, the Ctrl+R finder, and the Python catalog, all verified working.
- Making other LSP completion providers fuzzy (`#gh:` repos, models, directives, xprompt args stay prefix matched).
- Merging the two inventories into one loader. ACE warms a Python catalog off-thread; the LSP reads a launcher-written
  JSON catalog per request. Unifying them is a larger change than this plan and is not required for parity.

## Architecture

```
sase-core
  crates/sase_core/src/editor/completion.rs        collectors: fuzzy ranking, titles, bounds, pre-cap counts
  crates/sase_xprompt_lsp/src/server.rs            inventory: cached catalog, truncated_payloads
sase
  docs/editor.md                                   corrected reachability claims
```

`crates/sase_core/src/editor/completion.rs` is the whole of the first three phases. The shared matcher already exports
what they need: `fuzzy_match(query, text) -> Option<FuzzyMatch>` and `compare_fuzzy((&FuzzyMatch, &str), …) -> Ordering`
from `crates/sase_core/src/editor/fuzzy.rs`.

## Detailed Design

### Phase `rank` — fuzzy ranking in two collectors

`append_agent_page_candidates` and `append_artifact_index_candidates` keep their full enumeration (all agent
directories, all artifact-index rows — both already scanned in full today whenever the query matches nothing) and
replace the `starts_with` test with `fuzzy_match` against the payload. Collect every match, sort with `compare_fuzzy`,
then truncate to `ARTIFACT_REF_MAX_RESULTS`, returning the pre-cap count for phase `reach` to report.

Ranking before capping is what protects goal 4: a prefix match is tier 0, tier is the primary sort key, so any row a
prefix query reaches today sorts ahead of every fuzzy row and survives the cap.

Per-keystroke cost is unchanged in the worst case — a non-matching query already walks every entry — but the common case
now scores each candidate. `fuzzy_match` is one bounded pass per candidate over short strings (a global agent name, a
`<source>:<hex>` id), so this stays far inside the completion budget. Measure it against the real agent sidecar (5126
published agents on the machine this was verified on) and record the number.

Tests in `crates/sase_core/src/editor/completion.rs`: `@agent:` with a mid-name fragment surfaces a published page;
`@file:` with a fragment of the indexed file's name surfaces its id once phase `titles` lands, and with a digest
fragment before that; a prefix query still returns its row when more than `ARTIFACT_REF_MAX_RESULTS` rows fuzzy-match
ahead of it in walk order; existing `builds_agent_payload_candidates_from_published_pages` and
`builds_chat_and_indexed_file_payloads_but_not_remote_kinds` updated, not deleted — their prefix assertions are goal 4's
regression gate.

### Phase `titles` — real titles on payload rows

`artifact_ref_candidate(name, insertion, …)` sets `CompletionCandidate.name`, which `at_reference_payload_inventory`
maps to `AtReferencePayloadRowWire.label` and the menu maps to `AtReferenceRowWire.title`. Every collector currently
passes the payload for both. Give each kind the title ACE shows:

| Kind           | Title today | Title after                                                    |
| -------------- | ----------- | -------------------------------------------------------------- |
| document roles | payload     | frontmatter title, falling back to the basename                |
| `chat`         | payload     | file basename                                                  |
| `file`         | payload     | indexed file's basename (the index row already carries `path`) |
| `bead`         | payload     | bead title when the store makes it available, else the id      |
| `agent`        | payload     | short name (the segment after the last `.`), else the payload  |

Where a real title is not cheaply available, keep passing the payload — `at_reference_completion_item` already
suppresses an echoed title, so the fallback degrades to today's behavior rather than to noise.

Reading document frontmatter for up to `ARTIFACT_REF_MAX_RESULTS` files is the one costly entry in that table. Do it
after the cap, for surviving rows only, so the read is bounded by the row count and not by the corpus. That ordering
means a title-only match cannot pull a row past the cap in this phase; note the limitation in the phase's bead and let
phase `reach`'s cached inventory remove it by titling the whole cached corpus once.

Tests: a document row's `title` is its frontmatter title and differs from its payload; a `@file:` query matching only
the indexed basename returns the row; `labelDetails.detail` and the `documentation` second line are populated in
`crates/sase_xprompt_lsp/src/lsp_convert.rs` tests (today no fixture reaches them); an untitled document still falls
back to its payload and is still suppressed by `at_reference_completion_item`.

### Phase `reach` — bounds and disclosure

Two coupled changes.

**Reach.** Raise the enumeration bound so editors see what ACE sees. `bounded_relative_files` must stop conflating "how
many files to walk" with "how many rows to return": split `ARTIFACT_REF_MAX_RESULTS` (rows returned, stays 200 — an
editor completion list has no reason to be longer) from a new scan bound sized like ACE's `_MAX_DOCUMENT_ROWS_PER_KIND`
(5000). Walk up to the scan bound, match, rank, return the top `ARTIFACT_REF_MAX_RESULTS`, and report the pre-cap count.

**Cost.** That walk cannot run per keystroke. `load_artifact_ref_catalog` already re-reads and re-parses the
launcher-written catalog JSON on every request (deliberately, so launcher refreshes are visible without a restart), and
the payload walk would now sit on top of it. Add an in-process cache in `crates/sase_xprompt_lsp/src/server.rs` keyed by
catalog path plus mtime plus the resolved project, holding the parsed catalog and the enumerated, titled payload rows;
rebuild on mtime change or after a short TTL so external sidecar writes still appear. The
`sase.xpromptLsp.refreshCatalog` command should invalidate it explicitly. This is the LSP's equivalent of ACE's
off-thread warm plus memoized index.

**Disclosure.** Thread each collector's pre-cap count into `AtReferenceInventoryWire.truncated_payloads` so
`AtReferenceMenuWire.truncated_payloads` is finally non-zero when a bound bites. The response is already
`isIncomplete: true`, which is the protocol's own "there are more" signal, so no new field is needed on the wire — but
the count belongs in the row's `detail` or the list's own diagnostics rather than nowhere. Pick one and document it; the
requirement from the epic is that a cap the user cannot see must not read as "I searched everything."

Tests: a fixture root with more files than `ARTIFACT_REF_MAX_RESULTS` returns capped rows and a correct pre-cap count; a
fixture beyond the scan bound reports truncation; the cache returns identical rows on a second request without
re-walking (assert via a call counter or an mtime-stable fixture) and rebuilds after the catalog file's mtime changes;
the refresh command invalidates it.

### Phase `land`

- `docs/editor.md`: correct the reachability paragraph. State which kinds are enumerated and matched, that titles are
  matched, and what bound remains after phase `reach` with how it is disclosed. The current unqualified "any memorable
  fragment of its path or title" is the claim under repair — do not simply restate it.
- Publish the `sase-core` release containing phases `rank`, `titles`, and `reach`. release-plz owns `sase-core`
  versions; do not hand-edit them. Bump `sase-core-rs>=` in `pyproject.toml` only if a phase changed a Python-visible
  binding — none of them should, since the LSP ships as the separate `sase-xprompt-lsp` binary.
- Acceptance, against a freshly built LSP binary via `SASE_XPROMPT_LSP_CMD`, with the research sidecar opened:
  `@research:site` reaches `202607/sase_sites_hub_and_pages/sase_sites_hub_and_pages.md`; `@agent:` with a mid-name
  fragment reaches a published agent page; `@file:` with a fragment of an indexed file's name reaches its id;
  `labelDetails.detail` carries a title; a prefix query that works today still works.
- `sase-nvim`: extend `tests/lsp_artifact_ref_smoke.lua` to cover the `@agent:` fuzzy case, which is the
  client-observable half the Rust tests cannot prove.
- Verify: `just install`, `just check`, `just test`, `just test-visual` in `sase`; `cargo fmt --all -- --check`,
  `cargo clippy --workspace --all-targets -- -D warnings`, `cargo test --workspace` in `sase-core`.
- Close epic `sase-b3` with `sase bead close sase-b3 --note "<what was verified>"`, then run `just symvision` and remove
  any stale epic-symbol whitelist entries and unused code it reports for `sase-b3` — there are none today, so expect the
  pre-existing findings described under "Known unrelated failures" and leave them alone.
- Set `status: done` in the frontmatter of **both** this plan file and `plans:202607/fuzzy_artifact_ref_completion.md`.

## Verified State of `sase-b3` (do not redo)

All nine phases were re-verified before this plan was written, in the source and end to end, and are genuinely complete:
the shared matcher with its three-pass alignment and tier comparator, bundle-depth document discovery, the fuzzy menu
with match runs on the wire, the `AtReferenceInventory` pyclass, the LSP `filterText`/`isIncomplete` contract, per-role
Python catalogs with memoized indexes, path-first TUI rows with gold runs and an honest subtitle, and the Ctrl+R finder
on the shared matcher with the duplicate Python scorer deleted (`src/sase/core/fuzzy_facade.py` is now the only fuzzy
implementation in `src/`). `just test` passes 24171 / 7 skipped and `just test-visual` passes 392 / 1 skipped on the
epic's final commit. Nothing in this plan revisits that work.

## Known Unrelated Failures (do not fix here, do not be alarmed by)

- `just check` is red on `master` from 12 Symvision private-import findings under `src/sase/ace/tui/actions/clipboard/`,
  introduced by `df18f44f6 refactor(ace): split clipboard palette module` before this epic began. Every other gate —
  fmt, keep-sorted, ruff, mypy, pyscripts, changelog — passes. Fixing it means making 12 helpers public across four
  modules and belongs to its own bead.
- `sase-nvim` `tests/lsp_placeholder_smoke.lua` asserts document order for placeholder rows while `sase-core` has ranked
  live-before-literal since `641ca36`. Also its own bead.

## Risks & Mitigations

- **Prefix regression from ranking before capping.** Tier 0 sorts first by construction, and the two existing collector
  tests keep their prefix assertions as the gate.
- **Per-keystroke cost from a larger walk.** Phase `reach` pairs the raised bound with the cache in the same phase, on
  purpose — landing the bound without the cache would put a thousands-of-entries walk on every completion request.
- **A stale cache hiding a just-written document.** Keyed by mtime, bounded by a short TTL, and invalidated by the
  existing `sase.xpromptLsp.refreshCatalog` command, which is the same escape hatch the xprompt catalog already offers.
- **Frontmatter reads in phase `titles` costing more than they are worth.** Bounded to rows that survive the cap, so the
  read count is the list length and not the corpus size; the phase's fallback to the payload keeps an unreadable file
  from dropping a row.
