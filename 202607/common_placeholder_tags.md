---
tier: epic
title: Saved common placeholder tags in prompt completion
goal: 'Every `<foobar>` tag the user writes in a prompt is saved to a durable, capped,
  frecency-ranked store, and those saved tags appear beneath the current prompt''s
  own tags in the `<` completion menu, rendered in a visually distinct style that
  makes the two groups tellable apart at a glance.

  '
phases:
- id: core
  title: Placeholder candidate sources and common-tag input in sase-core
  depends_on: []
  size: medium
  description: '''Phase core — sase-core: candidate sources and common-tag input''
    section: extend the Rust placeholder completion engine so candidates carry a prompt/common
    source, accept a caller-supplied common-tag list, and emit prompt-local candidates
    before common ones; update the PyO3 binding, the xprompt LSP call sites, and the
    version contract.

    '
- id: store
  title: Durable common-placeholder store and prompt recording hook
  depends_on: []
  size: medium
  description: '''Phase store — durable common-placeholder store'' section: add a
    capped, atomically-written JSON store of placeholder tags ranked by use count,
    record tags from every submitted or abandoned prompt through the shared prompt-history
    choke points, and seed the store once from existing prompt history.

    '
- id: wiring
  title: Config field, warm cache, and completion menu wiring
  depends_on:
  - core
  - store
  size: medium
  description: '''Phase wiring — config, warm cache, and menu behavior'' section:
    add the ace.prompt_completion.common_placeholder_count config field, warm the
    store off-thread into an app-level cache, and thread common candidates through
    the placeholder menu''s open, refresh, and accept paths with the auto-versus-manual
    empty-prefix rule.

    '
- id: polish
  title: Distinct row styling, legend, and documentation
  depends_on:
  - wiring
  size: small
  description: '''Phase polish — distinct rendering, legend, and docs'' section: render
    common rows with their own badge and colour, add the two-source legend to the
    completion panel subtitle, add a PNG snapshot, and document the feature in ace.md,
    configuration.md, and the help popup.

    '
create_time: 2026-07-25 12:44:16
status: done
bead_id: sase-9m
---

- **PROMPT:** [202607/prompts/common_placeholder_tags.md](prompts/common_placeholder_tags.md)

# Plan: Saved common placeholder tags in prompt completion

## Background

`<foobar>` tags are called **placeholders** throughout this codebase and in `sase-core`. Today the completion menu for
them is built entirely from the text of the prompt currently being edited:

- `sase-core` `crates/sase_core/src/editor/placeholder.rs` scans a document for complete `<...>` spans
  (`scan_placeholder_spans`), detects the cursor context inside an unmatched `<`
  (`detect_placeholder_context_at_position`), and builds a deduplicated, case-insensitively prefix-filtered,
  document-ordered candidate list (`build_placeholder_completion_candidates`), excluding the span under the cursor.
- `src/sase/xprompt/placeholder_completion.py` is a typed facade over the `sase_core_rs` bindings
  (`placeholder_completion`, `placeholder_spans`).
- `src/sase/ace/tui/widgets/placeholder_completion.py` converts the Rust payload into `CompletionCandidate` rows and
  prompt-local character offsets.
- `src/sase/ace/tui/widgets/_placeholder_highlight.py` owns the per-document scan caches and
  `_placeholder_completion_at_cursor()`.
- `_file_completion_open.py` (`_try_auto_placeholder_completion`, `_open_placeholder_completion`),
  `_file_completion_tab.py` (`_try_file_completion_tab`), `_file_completion_refresh.py`, and
  `_file_completion_accept.py` drive the menu.
- `_prompt_input_bar_completion_rows.py::append_placeholder_completion_row` renders each row as a dim cyan `<> ` badge
  plus a cyan label; `_prompt_input_bar_completion_panel.py` sets the `placeholder` border title.

There is a close precedent for the "remembered across prompts" half of this feature: prompt-history **word** completion.
`ace.prompt_completion.history_word_count` caps it, `src/sase/history/prompt_words.py` derives words from prompt-history
shards, and `src/sase/ace/tui/actions/_startup_history_words.py` keeps an app-global cache warm off-thread with a cheap
shard staleness token. This plan deliberately mirrors that architecture where it fits and deliberately departs from it
where placeholders differ (see Design decisions).

## What the user asked for

1. Save every `<foobar>` tag written in a prompt, up to a limit set by a sase config field.
2. Show the saved tags when `<foobar>` completion is triggered.
3. Show them **beneath** the tags found in the current prompt.
4. Make the two groups visually distinct enough to tell apart at a glance.
5. The result must be intuitive, reliable, and beautiful.

## Design decisions

These are the decisions this plan commits to. An implementing agent should follow them rather than re-deriving them.

### D1 — Vocabulary: "placeholder", not "tag"

`tag` is already overloaded in this repo (tribe tags `%tribe:`, notification tags, VCS tags), while `<foobar>` is called
a _placeholder_ in the Rust engine, the Python facade, the completion kind, and the panel title. Code, config, and
internal docs therefore use **common placeholders**. User-facing prose in `docs/ace.md` introduces them as "the
`<foobar>` tags you have written before" so the user's own vocabulary is still discoverable.

### D2 — An authoritative store, not a derivation from prompt history

History-word completion derives its candidates from prompt-history shards on every rebuild. Placeholders do **not** do
that, for three reasons:

1. Ranking needs a **use count** ("common"), which requires scanning _all_ history rather than stopping at the newest N
   entries the way `collect_recent_prompt_words` does.
2. `is_recordable_prompt` skips prompts under five words, so history would silently drop tags from short prompts. The
   user asked to save _all_ tags they write.
3. A small dedicated file loads instantly, so the menu never needs the "loading…" placeholder row that the history-word
   menu requires.

So: a dedicated, authoritative JSON store, written at prompt-record time. Prompt history is used exactly once, as a
best-effort **seed** when the store file does not yet exist (D6), so the feature is useful on day one instead of after
weeks of use.

### D3 — Retention is LRU; display order is frecency

Two different orderings, deliberately:

- **Retention/eviction** (when the store exceeds the cap): drop the **least recently used** entries (`last_used`
  ascending). Evicting by count instead would mean that once the store is full a brand-new tag (count 1) is always the
  first candidate for eviction and could never establish itself. LRU retention avoids that pathology and matches how the
  rest of SASE ages prompt data.
- **Display order** (what the menu shows): `count` descending, then `last_used` descending, then `text` ascending. This
  is what "common" means to the user, and the final `text` tiebreak keeps ordering deterministic for tests.

The store is written already sorted in display order, so reads are a plain sequential parse with no re-ranking.

### D4 — Merge rules live in `sase-core`, storage lives in Python

Per the `rust_core_backend_boundary` memory: prefix matching, dedup, and candidate ordering are shared domain behaviour
that the xprompt LSP must eventually agree with, so `build_placeholder_completion_candidates` gains a
`common: &[String]` parameter and returns candidates that carry their source. Re-implementing a parallel
case-insensitive prefix filter in Python would fork the rule — exactly what the memory warns against.

Storage, by contrast, sits beside prompt history, which is already Python (`src/sase/history/prompt_store.py`). The
store therefore lives in `src/sase/history/`, and the ranked list is handed to core as a plain `list[str]` input. Core
stays pure and testable; it never reads disk.

The LSP passes an empty list in this epic, so LSP behaviour is unchanged. Wiring the LSP to a common-placeholder source
is explicitly out of scope.

### D5 — Auto-open stays quiet; manual `Ctrl+T` is exhaustive

`<` is common in prose and code (`a < b`, `<div>`, generics). Today, typing `<` with no other placeholder in the prompt
produces no menu at all, because the candidate list comes back empty. Adding saved tags would otherwise turn every stray
`<` into a popup of 100 unrelated entries — the single biggest way this feature could feel worse than no feature.

The rule:

- **Auto/soft trigger** (`_try_auto_placeholder_completion`): common placeholders are included only while the prefix
  after `<` is non-empty. A bare `<` behaves exactly as it does today.
- **Manual trigger** (`Ctrl+T`, `_try_file_completion_tab`): common placeholders are always included, even with an empty
  prefix. An explicit request gets the full list.

The rule is re-evaluated on every refresh, so backspacing from `<fe` to `<` in an auto-opened menu drops the common
group again, and typing a character brings it back. The trigger that opened the menu (`"auto"` or `"manual"`) is
recorded on the text area for the lifetime of the menu.

Note the interaction with `_try_file_completion_tab`'s existing "exactly one candidate auto-accepts" shortcut: with
common placeholders included, a `Ctrl+T` that matches exactly one saved tag now inserts it directly. That is consistent
with every other completion kind and is intended.

### D6 — One-time seed from prompt history

When the store file is absent, seed it off-thread by scanning prompt-history shards newest-first with
`iter_shard_paths_newest_first()`, extracting spans with the existing `placeholder_spans` binding, and counting
occurrences. Bound the work at the **24 newest shards** (shards are `YYMM`, so ~2 years) so a long history cannot turn
startup into a multi-second scan. Then write the store. If the seed fails for any reason, the store simply starts empty
— the feature degrades to "learns from here on", never to a broken menu.

The seed runs only when the file does not exist. It is not a repair mechanism for a store that exists.

### D7 — Failure is always silent and non-blocking

Recording tags happens on the prompt-submit path, which must never fail because of this feature. Every store write is
wrapped so that any exception is logged at debug level and swallowed. Reads treat a missing, empty, truncated, or
schema-mismatched file as an empty store and rewrite it on the next successful record. No user-visible error, no launch
failure, no menu breakage.

### D8 — Visual encoding

Two independent signals so the groups are distinguishable by colour _and_ by shape (which also survives monochrome-ish
terminals and colour-blind viewing):

| Group          | Badge | Style                                                    |
| -------------- | ----- | -------------------------------------------------------- |
| Current prompt | `<> ` | badge `dim cyan`, label `cyan` / `bold cyan` (unchanged) |
| Saved (common) | `★  ` | badge and label `#D7AF5F` / `bold #D7AF5F`               |

Both badges occupy **three cells** (`★` plus two spaces), so labels align in a single column regardless of group.
`#D7AF5F` is already the repo's muted-gold accent (`[private]` in `append_vcs_repo_completion_row`), so the menu stays
inside the existing palette rather than introducing a new hue.

Existing prompt-local rows are left byte-for-byte unchanged, so nothing about the current muscle memory shifts.

When — and only when — the visible menu contains both sources, the panel's `border_subtitle` shows a compact legend:
`<> prompt   ★ saved`. It teaches the encoding once, costs no row, and disappears when the menu is single-source. The
`border_title` stays `placeholder`.

### D9 — Dedup between the groups is exact-match

Core already dedups prompt-local candidates by exact string while filtering prefixes case-insensitively. Common
candidates are excluded when they exactly match a candidate already emitted from the prompt. Casing variants (`<Alpha>`
in the prompt, `alpha` saved) both appear — they are genuinely different insertions, and suppressing one would silently
withhold a candidate the user asked to save. Consistent, predictable, easy to test.

### D10 — `common_placeholder_count: 0` disables the whole feature

Zero means: do not record, do not load, do not display. The placeholder menu returns to exactly today's behaviour. This
is the escape hatch and must be tested as such.

## Configuration

New field under `ace.prompt_completion`:

```yaml
ace:
  prompt_completion:
    common_placeholder_count: 100
```

- Type `int`, minimum `0`, default `100`.
- Caps the number of entries retained in the store _and_ therefore the number that can be offered.
- Parsed with the existing `_parse_non_negative_int` helper in `src/sase/ace/tui/widgets/prompt_completion.py`.
- Read outside the TUI (by the recording path) through `sase.config.core.load_merged_config()`, which is already cached,
  so the prompt-submit path does no extra disk I/O.

Per the `gotchas` memory, `src/sase/default_config.yml` must be updated, and the JSON schema at
`src/sase/config/sase.schema.json` gains a matching property next to `history_word_count`.

## Store format

Path: `sase_home() / "prompt_placeholders.json"` (sibling of `prompt_history.json` and the `prompt_history/` shard
directory).

```json
{
  "version": 1,
  "placeholders": [
    { "text": "acceptance criteria", "count": 12, "last_used": "260725_143012" },
    { "text": "feature flag", "count": 4, "last_used": "260724_090511" }
  ]
}
```

- `version` is an integer; a payload whose `version` is unrecognised is treated as an empty store and replaced on the
  next write.
- `text` is the placeholder's inner text, exactly as written (no case folding, no trimming beyond what the Rust
  extractor already guarantees: non-empty, no leading/trailing whitespace, at most `PLACEHOLDER_MAX_INNER_CHARS`
  characters).
- `last_used` uses the same `generate_timestamp()` format as prompt history (`%y%m%d_%H%M%S`).
- Entries are persisted already sorted in display order (D3).
- Writes are atomic (tempfile plus `os.replace`) and serialised under an `fcntl` lock on a dedicated lock file,
  following the pattern already established by `save_shard` and `locked_prompt_history` in
  `src/sase/history/prompt_store.py`.

---

## Phase core — sase-core: candidate sources and common-tag input

Repo: the linked `sase-core` checkout. **Open it with the `/sase_repo` skill** (`sase repo open sase-core -r ...`) and
use the printed path for every read and write; do not reach for a sibling path directly.

### Rust changes — `crates/sase_core/src/editor/placeholder.rs`

1. Add a serialisable source enum and candidate struct:

   ```rust
   #[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
   #[serde(rename_all = "lowercase")]
   pub enum PlaceholderCandidateSource {
       /// Extracted from another `<...>` span in the document being edited.
       Prompt,
       /// Supplied by the caller from the durable common-placeholder store.
       Common,
   }

   #[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
   pub struct PlaceholderCandidate {
       pub text: String,
       pub source: PlaceholderCandidateSource,
   }
   ```

   Serde must emit `{"text": "...", "source": "prompt" | "common"}` — the Python facade rehydrates exactly that shape.

2. Change `PlaceholderCompletion::candidates` from `Vec<String>` to `Vec<PlaceholderCandidate>`.

3. Change the signature to
   `build_placeholder_completion_candidates(document, position, common: &[String]) -> Option<PlaceholderCompletion>` and
   extend the body so that, after the existing document-order pass:
   - Every entry of `common`, **in the order given by the caller**, is appended when it passes the same case-insensitive
     prefix filter and is not already present in the `seen` set (which by then holds the prompt-local texts). Common
     entries also dedup against each other through the same `seen` set.
   - Prompt-local candidates always precede common candidates. This ordering is a core guarantee, not a caller
     convention.
   - Passing an empty `common` slice must reproduce today's behaviour exactly.

4. Update `into_completion_list` so `detail` reflects the source (`"placeholder"` for prompt-local,
   `"saved placeholder"` for common) while `kind` stays `"placeholder"` for both — `kind` drives client behaviour and
   must not fork.

5. Update the existing unit tests for the new candidate type and add tests for:
   - common entries appended after prompt-local ones, in caller order;
   - a common entry equal to a prompt-local entry appearing exactly once;
   - common entries filtered by the same case-insensitive prefix rule;
   - an empty `common` slice producing byte-identical output to the pre-change behaviour;
   - `into_completion_list` carrying the per-source `detail`.

### Binding — `crates/sase_core_py/src/lib.rs`

- `placeholder_completion(text, line, character, common=None)` — a new trailing optional argument accepting a sequence
  of strings, defaulting to empty. Update the module docstring list at the top of the file.
- Extend `placeholder_bindings_return_plain_json_shapes` to cover the new candidate dict shape and the `common`
  argument.

### LSP — `crates/sase_xprompt_lsp/src/`

- Pass `&[]` at every `editor_build_placeholder_completion_candidates` call site (`server.rs`), and update
  `lsp_convert.rs::placeholder_completion_response` for the new candidate type. LSP output must be unchanged; its
  existing tests are the proof.

### Version contract

- Bump the `sase-core` workspace version in `Cargo.toml` from `0.9.1` to `0.9.2`.
- In this repo, raise the floor in `pyproject.toml` to `sase-core-rs>=0.9.2,<0.10.0` and confirm
  `tools/validate_sase_core_rs_version` passes.

### Acceptance

- `cargo test` passes in `sase-core`.
- `just install` in the sase workspace rebuilds `sase_core_rs` from the local checkout and `just check` passes here (the
  Python facade is updated in this phase only as far as needed to keep the existing callers importable — the behavioural
  wiring is phase `wiring`).

---

## Phase store — durable common-placeholder store

This phase is independent of phase `core` and can run in parallel with it. It must not import anything that depends on
the new Rust signature; it uses the existing `placeholder_spans` binding, which is unchanged.

### New module — `src/sase/history/prompt_placeholders.py`

Public surface:

- `common_placeholder_limit() -> int` — reads `ace.prompt_completion.common_placeholder_count` via
  `load_merged_config()`, clamped to `>= 0`, defaulting to `100`.
- `record_prompt_placeholders(text: str) -> None` — extracts spans from `text` with `placeholder_spans`, increments
  `count` and refreshes `last_used` for each distinct inner text (a tag repeated within one prompt counts once), inserts
  unseen texts with `count = 1`, evicts by LRU down to the limit (D3), and writes the store atomically under its lock.
  Returns immediately when the limit is `0`. Never raises (D7).
- `load_common_placeholders(limit: int) -> list[str]` — returns up to `limit` texts in display order. Missing, empty,
  corrupt, or version-mismatched files yield `[]`.
- `common_placeholder_source_token() -> tuple[str, int, int]` — `(path, st_mtime_ns, st_size)`, or a sentinel when the
  file is absent, mirroring `_shard_token` in `prompt_words.py` so the ACE cache can skip no-op rebuilds.
- `seed_common_placeholders_from_history(limit: int) -> bool` — the one-time seed of D6. Returns `False` immediately if
  the store file already exists or the limit is `0`. Otherwise scans at most the 24 newest shards from
  `iter_shard_paths_newest_first()`, accumulates counts, sets `last_used` from each entry's `last_used`, applies the
  same eviction and ordering rules, writes the store, and returns `True`.

Internal helpers (`_load_payload`, `_save_payload`, `_locked_placeholder_store`) follow the atomic-write and
`fcntl`-lock patterns already in `prompt_store.py`. Do not reuse `locked_prompt_history()` — a separate lock file keeps
prompt-history writes and placeholder writes from contending.

### Recording hook — `src/sase/history/prompt_store.py`

Record from the two functions that every prompt-submission path already funnels through:

- `add_or_update_prompt()` — call `record_prompt_placeholders(text)` **before** the `is_recordable_prompt(...)` early
  return, so short prompts still contribute their tags (D2, reason 2). Record once from the full text; multi-prompt
  segments need no separate call because span extraction already covers the whole string.
- `record_failed_launch_prompt()` — same, after the `text.strip()` guard.

Cancelled/abandoned prompts (`cancelled=True`, written when the prompt bar unmounts) do record their tags: the user
wrote them, and recovering a tag from a draft you abandoned is exactly the behaviour that makes this feature feel like
it remembers. Note this explicitly in the function docstrings.

Import `record_prompt_placeholders` lazily inside the functions, matching the existing lazy-import style in this module
(`from sase.agent.multi_prompt import ...`), to keep prompt-store import cost flat.

### Tests — `tests/history/test_prompt_placeholders.py`

Cover, using the existing prompt-history test fixtures for a sandboxed `sase_home`:

- a new tag is inserted with `count == 1`; recording the same tag again increments to `2` and advances `last_used`;
- a tag repeated three times in one prompt increments by exactly one;
- display order is `count` desc, then `last_used` desc, then `text` asc;
- eviction at the cap removes the least-recently-used entry, and a freshly recorded tag survives a full store;
- lowering `common_placeholder_count` prunes on the next write;
- `common_placeholder_count: 0` records nothing and loads `[]`;
- a truncated/invalid/version-mismatched file loads as `[]` and is replaced by the next successful write;
- a store write failure does not propagate out of `add_or_update_prompt` (patch the writer to raise);
- short prompts (under the five-word history threshold) still contribute tags while still not entering history;
- `seed_common_placeholders_from_history` populates counts from shards, respects the 24-shard bound, and is a no-op when
  the store already exists.

### Acceptance

`just install && just check` passes, and the new test module passes in isolation.

---

## Phase wiring — config, warm cache, and menu behaviour

Depends on `core` (new binding signature) and `store` (the load API). Start with `just install` so the rebuilt
`sase_core_rs` is present.

### Config plumbing

- `src/sase/ace/tui/widgets/prompt_completion.py`: add `common_placeholder_count: int = 100` to
  `PromptCompletionSettings` and parse it in `parse_prompt_completion_settings` with `_parse_non_negative_int`.
- `src/sase/default_config.yml`: add `common_placeholder_count: 100` under `ace.prompt_completion`.
- `src/sase/config/sase.schema.json`: add the property beside `history_word_count` (`type: integer`, `minimum: 0`,
  `default: 100`, with a one-line description).

### Python facade — `src/sase/xprompt/placeholder_completion.py`

- Add a frozen `PlaceholderCandidate` dataclass with `text: str` and `source: Literal["prompt", "common"]`.
- `placeholder_completion(text, line, character, common: Sequence[str] = ())` passes the list through to the binding.
- `_completion_from_dict` rehydrates `candidates` into `PlaceholderCandidate` values; an unrecognised `source` string
  falls back to `"prompt"` rather than raising.

### TUI result builder — `src/sase/ace/tui/widgets/placeholder_completion.py`

- Add a frozen `PlaceholderCompletionMetadata` dataclass carrying `source: Literal["prompt", "common"]`, attached as
  each `CompletionCandidate.metadata` (the same pattern `HistoryWordCompletionPlaceholder` and `JinjaCompletionMetadata`
  already use for row rendering).
- `build_placeholder_completion_result(text, cursor_offset, common=(), *, include_common_when_prefix_empty=False)`. Core
  always receives `common` and returns the merged, source-tagged list; when the payload's `prefix` is empty and
  `include_common_when_prefix_empty` is `False`, drop the `common`-sourced candidates in Python (D5). This is a filter
  on an already-computed list, not a second implementation of the matching rules (D4).
- Return `None` when the surviving list is empty, preserving today's "no candidates, no menu" contract.

### Warm cache — `src/sase/ace/tui/actions/_startup_common_placeholders.py`

A new mixin modelled on `_startup_history_words.py`, registered in `src/sase/ace/tui/actions/startup.py` and initialised
in `_state_init_late.py` beside the `_history_prompt_words_*` fields:

- `common_placeholders() -> list[str] | None` — warm cache, no disk access.
- `common_placeholders_generation() -> int` — a counter bumped on every successful publish, used as part of the
  per-document completion cache key.
- `warm_common_placeholders()` — off-thread staleness check against `common_placeholder_source_token()`, then
  `load_common_placeholders(limit)`. On the very first warm, if the store file is absent, call
  `seed_common_placeholders_from_history(limit)` first (D6) and load from the result. A limit of `0` publishes an empty
  list without touching disk.
- `_refresh_visible_common_placeholder_surfaces()` — reapplies a newly warm cache to any open placeholder menu,
  mirroring `_refresh_visible_history_word_surfaces`.

Call `warm_common_placeholders()` everywhere `warm_history_prompt_words()` is already called, plus after a prompt is
submitted, so a tag written in one prompt is offered in the next without restarting ACE.

**No "loading…" row.** If the cache is cold when a menu opens, the menu shows prompt-local candidates only and silently
gains the saved group when the cache lands. A spinner row inside a menu that already has real content would be noise,
and the store is small enough that a cold read is effectively instant.

### Menu plumbing

- `src/sase/ace/tui/widgets/_placeholder_highlight.py`: `_placeholder_completion_at_cursor()` takes an explicit
  `include_common_when_prefix_empty: bool` argument, reads the app cache (tolerating a missing app attribute, as
  `_file_completion_tab.py` already does for `history_prompt_words`), and extends the memo key from
  `(cursor_offset, len(text))` to `(cursor_offset, len(text), generation, include_common_when_prefix_empty)`. Getting
  this key wrong is the most likely source of a stale menu — call it out in the docstring.
- `src/sase/ace/tui/widgets/_file_completion_open.py`: `_try_auto_placeholder_completion` passes
  `include_common_when_prefix_empty=False` and records `self._placeholder_completion_trigger = "auto"` in
  `_open_placeholder_completion`.
- `src/sase/ace/tui/widgets/_file_completion_tab.py`: `_try_file_completion_tab` passes `True` and records `"manual"`.
- `src/sase/ace/tui/widgets/_file_completion_refresh.py` and `_file_completion_accept.py`: derive the flag from the
  stored trigger (`trigger == "manual"`) so refresh and accept always recompute against the same candidate set the user
  is looking at.
- Clear `_placeholder_completion_trigger` in `_clear_file_completion`, and declare the new attribute in
  `_file_completion_base.py` alongside the other completion state.
- `_structured_completion_claims_cursor()` keeps asking only "is there a placeholder context here" — it passes
  `include_common_when_prefix_empty=True` so a bare `<` with saved tags still shadows the prompt-word fallback, which is
  the existing precedence order for structured providers.

Accepting a common candidate reuses the existing placeholder accept branch unchanged: it already appends `>` when
`append_closing_bracket` is set and repositions the cursor past the bracket, and the replacement range comes from the
payload rather than from the candidate's origin.

### Tests

Extend `tests/ace/tui/widgets/test_placeholder_completion.py` and add cases for:

- common candidates appearing after prompt-local ones, in store order;
- an exact duplicate between prompt and store appearing once, attributed to the prompt;
- auto trigger with a bare `<` showing no common candidates, and showing them after one prefix character;
- manual `Ctrl+T` with a bare `<` showing the full saved list;
- backspacing from `<fe` to `<` in an auto-opened menu dropping the common group, and retyping restoring it;
- accepting a common candidate inserting `text>` with the cursor after `>`, both with and without a pre-existing closing
  bracket;
- refresh preserving the highlighted candidate across an edit;
- a cold cache producing a prompt-only menu that gains the saved group when the cache warms;
- `common_placeholder_count: 0` reproducing today's behaviour exactly (D10).

### Acceptance

`just install && just check` passes; the placeholder completion suite passes.

---

## Phase polish — distinct rendering, legend, and docs

Depends on `wiring`.

### Row rendering — `src/sase/ace/tui/widgets/_prompt_input_bar_completion_rows.py`

Rewrite `append_placeholder_completion_row` to branch on the candidate's `PlaceholderCompletionMetadata.source` per the
D8 table. Define the badges and the gold as module constants next to the function. A candidate with missing or
unrecognised metadata renders as the prompt variant, so nothing can render badge-less.

Keep both badges three cells wide (`"<> "` and `"★  "`) so labels line up down the menu.

### Legend — `src/sase/ace/tui/widgets/_prompt_input_bar_completion_panel.py`

In `show_file_completions`, when `is_placeholder` and the **visible** rows contain both sources, set
`panel.border_subtitle = "<> prompt   ★ saved"`; otherwise leave it empty. Compute this from `visible`, not from `rows`,
so the legend matches what is actually on screen while scrolling. `border_title` stays `placeholder`.

### Visual snapshot

Add a PNG snapshot to `tests/ace/tui/visual/` covering a placeholder menu that contains both groups, so the badge
alignment, the two colours, and the legend are pinned. Generate the golden with
`just test-visual --sase-update-visual-snapshots` and confirm `just test-visual` then passes clean. Inspect the rendered
PNG before accepting it: the point of this phase is that the two groups are obvious at a glance, and that is a judgement
only a human-visible artefact can settle.

### Documentation

- `docs/ace.md`: extend the prompt-completion section covering placeholder completion with how tags are saved, the cap,
  the auto-versus-manual empty-prefix rule (D5), and the visual encoding.
- `docs/configuration.md`: add `common_placeholder_count` to the `ace.prompt_completion` YAML sample and the field
  table, add a prose paragraph covering the store path, LRU retention versus frecency display order, the one-time seed,
  and the `0` disable, and add `src/sase/history/prompt_placeholders.py` to the section's `Source:` list.
- Per the `ace/CLAUDE.md` Help Popup Maintenance rule, review the `?` help modal and add or amend the placeholder
  completion entry if it describes placeholder completion behaviour. Respect the 57-character box width and the
  32-character keybinding description cap.
- No footer keybinding change: this feature adds no conditional keymap, so per the Footer Keybinding Convention it does
  not belong in the footer.

### Acceptance

`just install && just check` passes, `just test-visual` passes against the new golden, and `docs/` builds cleanly.

---

## Out of scope

- Wiring the xprompt LSP to a common-placeholder source (core accepts the input; the LSP passes an empty list).
- Any UI for browsing, editing, pinning, or deleting saved placeholders. If the user wants one later, the store's shape
  already supports it.
- Sharing the store across machines or projects; it is user-global, like prompt history.
- Changing prompt-history word completion, its config, or its cache.

## Risks and how this plan addresses them

| Risk                                                 | Mitigation                                                                              |
| ---------------------------------------------------- | --------------------------------------------------------------------------------------- |
| Every stray `<` in code or prose pops a 100-row menu | D5: auto-open requires a non-empty prefix before common tags join                       |
| A store write breaks an agent launch                 | D7: recording is best-effort, wrapped, and silent; atomic writes under a dedicated lock |
| The feature is useless until the store fills up      | D6: one-time bounded seed from existing prompt history                                  |
| New tags can never enter a full store                | D3: retention is LRU, not count-based                                                   |
| Stale menu after the cache warms or after a submit   | Generation counter in the completion memo key, plus a refresh hook for open menus       |
| Rust and Python disagree about prefix matching       | D4: all matching, dedup, and ordering rules stay in `sase-core`                         |
| Badge glyphs misalign across terminals               | D8: both badges padded to three cells; pinned by a PNG snapshot                         |
| Cross-repo version skew                              | `sase-core` bumped to `0.9.2` and the `pyproject.toml` floor raised to match            |
