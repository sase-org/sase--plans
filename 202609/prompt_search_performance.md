---
tier: tale
title: Make prompt search fast by removing repeated preprocessing and validation
goal: Substantially reduce fresh-process sase prompt search latency while preserving
  its matching, ranking, filtering, counts, and output contracts.
size: medium
proposed_by: bbugyi200.athena.0gh
status: done
---

# Prompt search performance

## Scope and tier

Implement this as one medium tale. The investigation identified concrete, bounded hot
paths; one coding agent can fix the adapters, integrate a small Rust archive reader, and
verify the complete command. A persistent search index, daemon, new CLI options, new
query language, and migration of the entire search engine are outside this plan.
Preserve existing search behavior and optimize the work required to produce it.

Open `sase-core` with `/sase_repo` before inspecting or changing that repository and use
only the returned checkout. Repository references below are relative to each repository
root. Shared archive discovery and parsing belong in `crates/sase_core`; Python retains
presentation and thin binding/offset adapters. The existing Rust fence and inline-code
scanners remain authoritative.

## Evidence collected before proposing

Read-only profiling used this sase checkout's Python sources with the installed SASE
runtime. The checkout's virtualenv could not import its editable Rust extension; do not
mistake that environment issue for a search regression. No implementation files were
changed. Temporary function replacements existed only inside benchmark processes.

The observed corpus contained 4,980 archive Markdown paths (15,765,141 bytes), three
local history shards (4,880,897 bytes), and zero artifact payload files. Search loaded
4,973 usable archive hits and 7,821 local records, yielding 12,794 combined hits. Counts
can change as normal agent activity continues.

Findings:

1. `src/sase/xprompt/_fenced_blocks.py` reconstructs a full UTF-8 byte-to-character
   dictionary in `_byte_range_to_character` for each range of every non-ASCII fence.
   There are up to five ranges per fence, and both title and tag extraction invoke this
   path. This multiplies document length by fence count. A 12-second truncated cProfile
   run had loaded only 372 archive documents; 7.49 seconds were attributed to 799
   `_byte_to_character` calls. Another profiled run was interrupted after more than a
   minute in this conversion loop. These are profiler timings, not fresh-command latency
   measurements.
2. `src/sase/xprompt/_inline_code.py` separately constructs complete forward and reverse
   maps for non-ASCII documents. After experimentally reusing fence maps, inline
   conversion became the largest remaining profiled contributor.
3. `src/sase/prompt/search/sources.py` calls `clean_prompt_preview` for the title and
   `summarize_prompt_for_list` for tags. Both protect literal zones and scan references;
   the latter already computes the cleaned preview.
4. `list_prompt_archive_files` in `src/sase/agents_sync/prompt_archive/validation.py`
   calls `validate_prompt_archive(...).files`. Inventory therefore checks links, walks
   artifacts, and hashes matching artifact filenames, although search discards the
   diagnostics. Search subsequently rereads and reparses each prompt in
   `_load_archive_file`. There were no artifact payloads in the measured archive, so
   artifact hashing is an additional scaling problem, not an explanation for this
   particular corpus's delay.
5. `_render_full_local` in `src/sase/prompt/cli_search.py` calls
   `resolve_prompt_selector` for every shown local hit. Each call reloads and hashes the
   entire local history. Its repeated I/O is unnecessary because the search already
   loaded those records.
6. The pure matching engine is comparatively inexpensive. An unprofiled experiment using
   sparse offset conversion and one summary per prompt collected the combined corpus in
   3.264 seconds and matched `prompt` in 0.119 seconds, reporting 5,657 matches. Reusing
   whole-document offset maps instead took about 5.788 seconds to collect and 0.084
   seconds to match. The unchanged unprofiled collector exceeded a 12-second cutoff; its
   complete elapsed time was not measured. The experiments retained archive validation
   and duplicate reads. These demonstrate promising directions, not production
   correctness or a completed end-to-end speedup measurement.

## Implementation

Before editing implementation code, capture a bounded baseline on the fixed benchmark
corpus described below. Preserve enough expected output to compare ranked results and
metadata after the optimization. Keep benchmark setup outside production stores; do not
migrate, rewrite, or prune the user's history to make the measurement easier.

### 1. Make Python/Rust offset conversion proportional to the input

Update `_fenced_blocks.py` and `_inline_code.py`, sharing a small private Python UTF-8
offset adapter if useful. This is representation conversion; do not move fence discovery
or inline delimiter semantics out of Rust.

For fenced details, collect all requested byte endpoints across all rows and convert
them together. For non-ASCII text, sort/deduplicate endpoints and count characters over
successive, disjoint UTF-8 slices. Encode the document once and use C-backed
encode/decode operations over disjoint slices rather than a Python loop and dictionary
entry for every character. Convert all five detail ranges using that single mapping. A
range-only consumer need not construct unused structured details if the existing binding
exposes the necessary primitive.

For inline code, translate normalized character-mask endpoints in one ordered batch,
call the existing Rust scanner, then convert only its returned byte endpoints in another
batch. Do not allocate a full character-to-byte list plus a full inverse dictionary.
Short-circuit empty results and retain the ASCII identity fast path.

Target conversion complexity is O(B + R log R), where B is document bytes and R is the
requested endpoints, with O(R) offset metadata rather than O(characters) Python objects.
Do not independently decode every prefix from byte zero. Preserve ordering, exact Python
character offsets, EOF endpoints, optional absent detail fields, mask clipping, and
existing handling of unexpected wire values. Keep all maps scoped to one call; do not
introduce a global prompt-body cache or retain historical prompt text in long-lived TUI
processes.

### 2. Derive search title and tags from one lexical pass

In `sources.py`, obtain cleaned preview and body tags together for each source record.
Reuse the common scan behind `summarize_prompt_for_list` and `clean_prompt_preview`;
avoid invoking two complete pipelines. Prefer a focused shared extraction helper if
computing unrelated list-row VCS/project columns remains material in the profile.
Existing presentation helpers should delegate to that shared extraction where
appropriate, not acquire a second reference parser.

Preserve the existing title fallback, tag normalization and stable deduplication,
workflow-reference exclusion, and frontmatter/body tag merging. In particular, workflow
metadata lookup failure must not discard an otherwise available cleaned title: keep
lexical extraction independent of optional workflow classification or retain the
equivalent rare-error fallback. Cache reuse, if any, must be local to one record/search,
not keyed globally by prompt text or stale workflow config.

### 3. Separate prompt inventory from archive validation and read each file once

Add a small typed read-only archive-document API in sase-core, exposed through
`crates/sase_core_py` and a thin Python facade. Reuse the existing Rust
`plan/artifact_link.rs` header parser. Do not reuse plan search, whose corpus, ranking,
and date semantics differ from prompt search.

The inventory should discover the same sorted `prompts/*/*.md` files, preserve README
exclusions and the optional month selector, and read/parse each document once. Return
enough of the document snapshot to derive existing inventory and search results without
reopening it: relative path/locator, original content for existing frontmatter
extraction, clean body, parsed header sections, and per-file read/parse outcome. Use
explicit typed wire data and schema checks consistent with the surrounding binding code.
Missing roots, unreadable/invalid-UTF-8 files, and invalid header dispositions must
retain the existing caller-specific behavior rather than converting a tolerable file
error into a whole-search failure.

Have `list_prompt_archive_files` and `load_archive_prompt_hits` adapt this inventory.
The explicit `validate_prompt_archive` path should consume the same parsed snapshots and
then perform its existing plan/link/artifact/manifest checks. Preserve its diagnostic
codes and inventory fields. Search and inventory must never walk/hash artifact payloads,
check counterpart repositories, or run validation merely to count an ARTIFACTS section.
Do not remove any explicit validation capability.

Retain existing frontmatter handling, clean-body stripping, recorded SHA precedence and
fallback hashing, date resolution, plan labels, and artifact counts. Do not use
directory month pruning as a substitute for resolved dates: frontmatter dates can
disagree with directory names. Do not add persistent inventory caching in this tale;
every invocation must see additions, edits, renames, and deletions immediately.

### 4. Render full local matches from the loaded records

Carry the original `PromptHistoryRecord` or equivalent immutable rendering metadata
through local hit adaptation. The original creation `timestamp` is required in addition
to `last_used`; do not reconstruct it from the search date. Reuse
`render_prompt_markdown` with that snapshot and eliminate per-hit global
`resolve_prompt_selector` calls. Internal model fields may have defaults for existing
synthetic callers, but keep the JSON renderer's public schema unchanged. Keep this
rendering context out of equality/ranking decisions and the JSON wire payload; carry
references or the missing scalar metadata rather than copying large prompt bodies. If a
legacy/internal caller lacks rendering metadata, resolve missing records at most once
for the render operation, not once per hit.

All normal CLI search formats should load local history at most once. A file changing or
disappearing after collection should still permit rendering the already selected
snapshot. Preserve exact body/newline behavior and markdown metadata parity with
`prompt show` on an unchanged store.

### 5. Preserve the search contract

Keep `search/engine.py` behavior and the CLI argument surface stable. Do not speed up
the command by stopping after 20 matches, searching only previews, reducing source
coverage, or replacing literal substring matching with token matching.

The result remains Unicode `casefold` substring matching across title, body, id, path,
plan, and tags, with stable matched-field order. Preserve archive-first ranking,
strong-field tier, recency and locator tie breaks, accurate pre-limit source counts,
nonpositive unlimited limits, inclusive date bounds, OR across tag filters, and
cancelled-only filtering of local hits. Preserve source alias `sdd`, all-source
tolerance of an unavailable archive, and explicit archive-source errors.

Cross-source SHA deduplication happens before query/date/tag/cancelled filtering;
archive hits suppress local duplicates and retain `also_in_local`. Do not push filters
into loading in a way that changes this precedence. Multiple archive paths with the same
digest retain current behavior. Keep the existing sharded local-history reader,
duplicate reconciliation, and legacy migration semantics.

## Verification and acceptance

Read the project's lint/test, xprompt, and TUI performance guidance before
implementation. Read linked sase-core instructions after opening it. Bootstrap this
checkout with `just install` as needed and ensure its rebuilt binding is used; do not
validate the implementation against an unrelated installed wheel.

Extend meaningful coverage rather than relying on elapsed-time assertions in ordinary
tests:

- Offset and literal-zone parity: Unicode before/inside/after ranges (emoji, combining
  characters, and multibyte letters), ASCII, CRLF, empty input, adjacent/nested-looking
  and unclosed fences, optional info/closing ranges, inline masks and EOF. Assert both
  detail ranges and protect/unprotect behavior. Add a large Unicode/many-fence
  regression with deterministic work bounds that catches rebuilding/scanning the whole
  document per endpoint. Check masks and returned ranges against a simple independent
  reference on small randomized inputs. Relevant suites include
  `tests/test_preprocessing_code_blocks.py`, `tests/test_xprompt_inline_code.py`, and
  existing prompt metadata tests.
- Search metadata: one shared extraction per ordinary prompt, unchanged titles and tags
  for fences, inline code, directives, namespaced references, custom workflow names,
  duplicate tags, and extraction-error fallbacks. Do not expand or execute historical
  prompt references while deriving metadata.
- Archive core/binding/facade parity: canonical layout, month selection, malformed and
  missing files, frontmatter dates/digests/tags, header dispositions, and exact
  inventory fields. Assert one prompt read/header parse per inventory/search and zero
  artifact-file reads or counterpart validation. Include an artifact-heavy fixture that
  still produces the same explicit validator diagnostics for missing links, digest
  mismatches, and orphans.
- Retain
  `tests/prompt_command/test_search_{engine,sources,dates,cli,render,large_history}.py`
  coverage. Add dedup-before-filter adversaries and body-tail-only matches in large
  pastes. Check JSON payloads, ranked IDs, matched fields, counts and full markdown, not
  just whether a query finds something.
- Full output: count history loads for 1, 20, and unlimited local hits. Ordinary search
  must load history once and never resolve each hit globally; original timestamps and
  exact prompt bodies must survive rendering.

Add a repeatable developer benchmark using disposable synthetic stores, with roughly
5,000 archived prompts and 8,000 local entries, multilingual long pastes, many code
fences, metadata-only hits, cross-source duplicates, and artifact-heavy variants. Keep
synthetic corpus size/seed explicit and print aggregates rather than personal prompt
contents. Run commands in fresh processes with output captured so terminal speed does
not dominate. Benchmark default compact, JSON, and full output, each source selection, a
common query, a rare body-tail query, and a no-match query. Include a Unicode/many-fence
microbenchmark at increasing sizes. Report import/root resolution, archive loading,
local loading, matching, rendering, elapsed time, and peak memory separately where
practical.

Capture baseline and final medians over at least three comparable runs on the same fixed
corpus/runtime, distinguishing OS-cached runs from a first invocation; do not flush
host-wide caches. Use bounded timeouts for the old implementation and label incomplete
baselines as lower bounds. Profiling adds overhead and is for attribution, not the
speedup denominator.

Acceptance targets: at least 5x lower fresh-process default-search median on the
representative Unicode/fenced corpus, with a target of 2 seconds and a required 3-second
median budget for approximately the measured corpus size on this host. Demonstrate at
least 10x faster offset conversion on the many-fence Unicode microbenchmark, with
near-linear scaling as text and fence count grow together. The measured 3.264-second
prototype still included archive validation and duplicate I/O, so the final budget
requires all planned fixes. If the budget is missed, reprofile and finish the remaining
attributable work within this scope before claiming success; do not hide regressions by
narrowing the corpus or query. Unlimited full output is naturally proportional to output
bytes; require one history load and eliminate repeated preprocessing rather than
imposing the same wall-clock budget on an unbounded dump.

Run focused correctness tests, `just check` in sase, and sase-core's required
`just check`/`scripts/check.sh` including PyO3 tests. Because the shared offset adapters
affect launch parsing and TUI highlighting, run relevant existing highlight/literal-zone
tests and the required exhaustive landing gate through `/sase_monitor`, including
`just check-full` as prescribed by project guidance. Inspect visual output if a snapshot
fails; this optimization should preserve it. Record final benchmark measurements and
correctness/check results in the change summary. Host-owned finalization handles commits
and landing.
