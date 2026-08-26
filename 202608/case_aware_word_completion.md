---
tier: tale
title: Make prompt-history word completion case-smart
goal:
  Ctrl+T word completion infers the casing the user typed instead of pasting the stored
  spelling verbatim, and collapses case-variant spellings of the same word into one row.
size: medium
proposed_by: bbugyi200.athena.0e3
create_time: 2026-08-26 07:58:34
status: wip
---

# Plan: Case-smart prompt-history word completion

## Goal

Typing `SPECTAC` and pressing `<ctrl+t>` should insert `SPECTACULAR` when the only
matching history word is `spectacular`. More generally, the `prompt_word` and
`history_word` completion menus should:

1. infer the casing the user typed and apply it to the completed word, and
2. treat case-variant spellings of the same word (`xprompt` / `Xprompt` / `XPROMPT`) as
   one completion, not three competing rows.

## Background: what already works, and what does not

Matching is **already** case-insensitive, so that part needs no work:

- `PromptWordIndex.word_ids_with_prefix()` bisects a `casefold()`-sorted key array
  (`src/sase/history/prompt_word_index.py`).
- `rank_recent_history_words()` and `build_history_word_completion_result()` both filter
  with `word.casefold().startswith(prefix.casefold())`.
- `shared_word_extension()` compares candidates case-insensitively.

What is broken is everything _downstream_ of matching: the candidate's **stored
spelling** is used verbatim as the insertion, and each distinct spelling is its own row.
All four gaps below were reproduced against the real prompt-history corpus (7,454
prompts / 10,017 distinct words) with `min_length=5`:

| #   | Input                                         | Today                                             | Wanted                        |
| --- | --------------------------------------------- | ------------------------------------------------- | ----------------------------- |
| 1   | `SPECTAC` + history `spectacular`             | inserts `spectacular`                             | `SPECTACULAR`                 |
| 2   | `XPROMP` + `<ctrl+t>` (shared-extension path) | inserts `XPROMPt`                                 | `XPROMPT`                     |
| 3   | `readm` + `<ctrl+t>`                          | inserts `readmE` (extension taken from `READMEs`) | `readme` / `README`           |
| 4   | `githu`                                       | 9 rows including `github`, `GitHub`, `Github`     | 7 rows, one per distinct word |

Gaps 2 and 3 come from `_try_history_word_completion_tab()` /
`_try_prompt_word_completion_tab()` in
`src/sase/ace/tui/widgets/_file_completion_tab.py`, which insert
`f"{result.prefix}{result.shared_extension}"` — the typed prefix keeps its casing while
the extension is sliced out of an arbitrary candidate's stored spelling, so the two
halves disagree.

Gap 4 is corpus-wide, not incidental: **551 fold groups hold more than one spelling, and
606 of the 10,017 word rows are redundant case variants.** It also blocks the goal
example directly. The real corpus contains both `SPECTACULAR` and `spectacular`, so
`SPECTAC` opens a two-row menu today; only after collapsing them into one row does
`<ctrl+t>` take the single-candidate auto-accept branch and complete in one keystroke,
which is the behavior the goal asks for.

## Design

### D1. The casing policy

Add one pure helper, used by every word-completion surface:

```python
def apply_word_case(canonical: str, prefix: str) -> str:
    """Return *canonical* re-cased to honor the casing typed in *prefix*."""
```

Precondition: `canonical.casefold().startswith(prefix.casefold())`.

Rules, in order:

1. **Fold-align.** Find the smallest `split` where
   `canonical[:split].casefold() == prefix.casefold()`. Casefolding can change length
   (`"Straße".casefold() == "strasse"`), so a prefix can straddle a character and have
   no aligned split. When there is no aligned split, return `canonical` unchanged —
   except under rule 2, which may still uppercase. `remainder = canonical[split:]`.
2. **SHOUT** — the prefix has at least two cased characters and every cased character is
   uppercase: return `prefix + remainder.upper()`. With no aligned split, return
   `canonical.upper()` if it still casefold-starts with the prefix, else `canonical`.
   The two-character floor keeps a single leading capital (`S`, `I`) from being read as
   shouting.
3. **Intrinsic casing wins** — `canonical` has an uppercase character anywhere after its
   first cased character (`GitHub`, `README`, `NASA`, `iPhone`): return `canonical`
   unchanged. This is today's behavior and it is worth keeping: it is what makes typing
   `githu` insert `GitHub`.
4. **Otherwise** return `prefix + remainder`. This preserves exactly what the user typed
   and takes the rest from history: `Spectac` -> `Spectacular`, `S` -> `Spectacular`,
   `als` + stored `Also` -> `also`.

Worked examples (all verified against a prototype of this rule):

| prefix    | canonical     | result        | rule                 |
| --------- | ------------- | ------------- | -------------------- |
| `SPECTAC` | `spectacular` | `SPECTACULAR` | 2                    |
| `spectac` | `spectacular` | `spectacular` | 4                    |
| `Spectac` | `spectacular` | `Spectacular` | 4                    |
| `S`       | `spectacular` | `Spectacular` | 4                    |
| `als`     | `Also`        | `also`        | 4                    |
| `githu`   | `GitHub`      | `GitHub`      | 3                    |
| `GITHU`   | `GitHub`      | `GITHUB`      | 2                    |
| `readm`   | `README`      | `README`      | 3                    |
| `nas`     | `NASA`        | `NASA`        | 3                    |
| `sPectac` | `spectacular` | `sPectacular` | 4                    |
| `stras`   | `Straße`      | `Straße`      | 1 (no aligned split) |

Rule 3 deliberately beats rule 4 but loses to rule 2: shouting `GITHU` yields `GITHUB`,
and shouting a snake_case identifier yields `AGENT_CHAT_FROM_NAME`. That is the honest
consequence of an explicit all-caps prefix and should be documented in the helper's
docstring, not special-cased away.

Home for the helper: `src/sase/ace/tui/widgets/prompt_word_completion.py`, next to
`shared_word_extension()` and `is_word_character()`, which the history-word module
already imports. Keep it a pure function with no widget or index dependency so the
ranking layer can use it too.

### D2. Recase the shared extension

`shared_word_extension(insertions, prefix)` currently slices the shared run out of
`insertions[0]`, so its casing comes from an arbitrary candidate. Change it to return
the extension already re-cased against `prefix` using the same policy: compute the
shared run as it does now, then hand `prefix + shared_run` through `apply_word_case()`
and return everything past `len(prefix)`. Keep the existing guard that bails to `""`
when the shared run does not casefold-align with the typed prefix.

Both `_try_prompt_word_completion_tab()` and `_try_history_word_completion_tab()` then
keep writing `f"{result.prefix}{result.shared_extension}"` unchanged and get `XPROMPT`
instead of `XPROMPt`. Both already rebuild the result from the new cursor position
afterwards, so the second `<ctrl+t>` sees the new, longer prefix and stays consistent.

### D3. Collapse case variants into one candidate

Give `PromptWordIndex` one word id per casefold key instead of one per spelling, so
every downstream consumer (`mru`, `document_frequency`, `last_used_epoch`, postings,
relation context) sees a single merged word.

Do this as a **post-pass over distinct words**, not inside the tokenizing loop. The
interning loop runs once per token occurrence; the post-pass runs once per distinct
word. Measured on the real corpus, the post-pass costs ~5 ms against a ~438 ms total
index build, and the build already runs in a background warm worker
(`warm_history_prompt_words`), so this is inside budget. Casefolding inside the
interning loop instead would multiply the cost by the occurrence count for no benefit —
do not do it that way.

The post-pass must:

- Group word ids by `word.casefold()`.
- Pick a **canonical spelling** per group: highest `document_frequency`, then most
  recent `last_used_epoch`, then a deterministic tiebreak (lowest word id). Verified on
  the real corpus, this picks `GitHub` over `github`/`Github`, `README` over `readme`,
  and `spectacular` over `SPECTACULAR`.
- Merge each group's postings (union of prompt ids, deduplicated and kept ascending),
  recompute `document_frequency` from the merged postings, and take the group's maximum
  `last_used_epoch`. Merging matters for ranking quality, not just row count: a word
  split across three spellings currently ranks as three weak rows instead of one strong
  one.
- Remap `prompt_words` rows to canonical ids and re-deduplicate within each prompt.
- Rebuild `folded_keys` / `folded_order` / `mru` / `max_document_frequency` from the
  collapsed word list.

`word_ids_for_spelling()` now returns at most one id and keeps its signature. Its
callers in `_context_word_key()` still work, and prompt context matching becomes
case-insensitive as a side effect — a small relation-quality win, worth calling out in
the module docstring.

Also fold-collapse the non-indexed fallback builder
`build_history_word_completion_result(text, cursor, words)`: change its `seen` set from
exact spellings to casefolds, keeping the first (most recent) spelling as canonical.
That path serves callers and test harnesses with no index provider.

### D4. Keep canonical spelling and insertion separate

Once the insertion is re-cased it is no longer the stored spelling, and two call sites
break if they cannot tell the difference:

- **Deletion.** `_delete_selected_history_word_completion()` in
  `_file_completion_accept.py` persists `selected.insertion` through
  `delete_prompt_word()`. Persisting `SPECTACULAR` when history stores `spectacular`
  would silently no-op, because `prompt_word_deletions` matches exact spellings.
- **Selection preservation.** `_replace_completion_candidates_preserving_selection()` in
  `_file_completion_refresh.py` re-finds the highlighted row by `insertion`. If the
  applied casing shifts between keystrokes the highlight resets to row 0.

Fix both by carrying the canonical spelling in `CompletionCandidate.name` (already
present, and unused for word candidates — it is only read for file/xprompt sorting) and
the re-cased text in `display` and `insertion`. Then:

- The delete path passes `selected.name`, not `selected.insertion`.
- Selection preservation for word kinds matches on `name`.
- Deletion matching becomes fold-aware: compare `word.casefold()` against a folded view
  of the deleted set in `_prefix_candidate_word_ids()`, `rank_recent_history_words()`,
  `_mru_words_from_index()`, and `forget_history_prompt_word()`. **This is an
  intentional behavior change**: deleting the one collapsed row now suppresses every
  case variant of that word, which is the only coherent meaning once the row _is_ the
  whole fold group. Keep persisting the canonical spelling so existing
  `prompt_word_deletions.json` entries keep matching their own fold; no store migration
  is needed.

Set `display` to the re-cased insertion so the menu is WYSIWYG. Row rendering in
`_history_word_rows.py` and `_prompt_input_bar_completion_rows_simple.py` reads
`candidate.display` and needs no change.

### D5. Suppress rows that would be a no-op

`build_*_history_word_completion_result()` and `build_prompt_word_completion_result()`
currently pass `exclude_exact=prefix` and drop a candidate whose stored spelling equals
the typed prefix (when there is no right-hand suffix), because accepting it would change
nothing. Under the new policy the right test is on the **computed insertion**, not the
stored spelling: replace the `word == exclude_exact` comparison with "drop the candidate
when its re-cased insertion equals the typed prefix and there is no word suffix."

This is what makes the goal example land: the corpus contains `SPECTAC` itself as a word
(the user typed it), so the `SPECTAC` prefix yields two collapsed groups — `spectac` and
`spectacular`. The `spectac` group re-cases to `SPECTAC`, equals the prefix, and is
dropped, leaving exactly one candidate, which `<ctrl+t>` auto-accepts as `SPECTACULAR`.

It also fixes a real case the old rule missed: typing `github` when history stores
`GitHub` currently offers a row, and under a naive prefix-preserving policy that row
would insert `github` and do nothing. Under rule 3 it inserts `GitHub` — a useful case
correction — and correctly stays in the menu.

`rank_history_words()` / `rank_recent_history_words()` take `exclude_exact` today and
are only called from `history_word_completion.py`, so either thread the policy into
ranking or drop the parameter and filter in the builder. Prefer filtering in the
builder, where both the prefix and the insertion are already known; at most one
candidate is ever dropped, so the interaction with `limit` is not worth extra machinery.

### D6. Apply the same policy to prompt-local word completion

`build_prompt_word_completion_result()` (words earlier in the _current_ prompt) shares
the `<ctrl+t>` key, the `WordCompletionResult` shape, `shared_word_extension()`, and
`_commit_word_completion()` with the history path — and it runs _first_, falling back to
history only on a miss. Leaving it case-naive would mean `SPECTAC` completes differently
depending on whether the word happens to appear earlier in the prompt you are typing.
Apply D1, D2, D4 (`name` = canonical), and D5 there too, and fold-collapse its
`latest_offset` dict so `Also` and `also` earlier in the same prompt produce one row.

## Implementation steps

1. **`src/sase/ace/tui/widgets/prompt_word_completion.py`** — add `apply_word_case()`
   (D1) with the rule order and the shout/intrinsic definitions in its docstring; rework
   `shared_word_extension()` to return a re-cased extension (D2); fold-collapse and
   re-case `build_prompt_word_completion_result()`, setting `name` to the canonical
   spelling and `display`/`insertion` to the re-cased text, and switch its exact-match
   suppression to the insertion-equals-prefix rule (D5, D6).
2. **`src/sase/history/prompt_word_index.py`** — add the fold-collapse post-pass to
   `_build_prompt_word_index_from_paths()` (D3). Document that `PromptWordIndex.words`
   now holds canonical spellings, one per fold group.
3. **`src/sase/history/prompt_word_ranking.py`** — make the `deleted` check fold-aware
   in `_prefix_candidate_word_ids()` and `rank_recent_history_words()`; remove or
   retarget `exclude_exact` per D5.
4. **`src/sase/ace/tui/widgets/history_word_completion.py`** — re-case candidates in
   both `build_history_word_completion_result()` and
   `build_indexed_history_word_completion_result()`, set `name` to the canonical
   spelling, fold-collapse the MRU fallback's `seen` set, and apply the D5 suppression.
5. **`src/sase/ace/tui/widgets/_file_completion_accept.py`** — pass `selected.name` to
   `delete_prompt_word()` and to `forget_history_prompt_word()`.
6. **`src/sase/ace/tui/widgets/_file_completion_refresh.py`** — preserve selection by
   `name` for the two word kinds.
7. **`src/sase/ace/tui/actions/_startup_history_words.py`** — make
   `_mru_words_from_index()` and `forget_history_prompt_word()` fold-aware for
   deletions.

## Testing

Unit (pure, fast — put the bulk of the coverage here):

- `tests/ace/tui/widgets/test_prompt_word_completion.py` — table-drive
  `apply_word_case()` over every row of the D1 example table, plus the `ß`
  no-aligned-split case and a caseless prefix (`x-1`). Add a property-style assertion
  that the result always casefold-starts with the typed prefix _or_ is exactly
  `canonical`.
- Same file — `shared_word_extension()` returns `TAC` for prefix `SPEC`, `tac` for
  `spec`, and `""` when the shared run does not align.
- `tests/history/test_prompt_word_index.py` — a shard containing `Also`, `also`, `ALSO`
  yields one word id whose `document_frequency` is the merged count and whose spelling
  is the canonical pick; `word_ids_for_spelling("ALSO")` returns that id. Update
  `test_prefix_lookup_is_case_insensitive_and_uses_unicode_folding`, which asserts
  `index.words.index("Alpha")` and today's two-spelling behavior.
- `tests/history/test_prompt_word_ranking.py` — a deleted `Also` suppresses `also`.
- `tests/ace/tui/widgets/test_history_word_completion_builder.py` — update
  `test_history_builder_filters_casefold_prefix_in_mru_order`, which currently asserts
  three separate `ALPINE`/`Alpha`/`alphabet` rows and `shared_extension == "P"`.

Widget/pilot, using `RankedHistoryCompletionTestApp` and `seeded_index()` in
`tests/ace/tui/widgets/_history_word_completion_helpers.py`:

- The goal example end to end: seed `spectacular`, type `SPECTAC`, press `<ctrl+t>`,
  assert the text is `SPECTACULAR` and the menu closed (single-candidate auto-accept).
- Seed both `spectacular` and `SPECTACULAR`; assert one row, and that `<ctrl+t>` still
  auto-accepts.
- Shared-extension path: seed `xprompt` and `xprompts`, type `XPROMP`, assert `XPROMPT`
  (not `XPROMPt`).
- Intrinsic casing: seed `GitHub`, type `githu`, assert `GitHub`.
- `tests/ace/tui/widgets/test_history_word_completion_delete.py` — `<ctrl+d>` on a row
  reached through an upper-case prefix persists the **canonical** spelling and removes
  every case variant from the menu.

## Non-goals

- **No new config field and no feature flag.** `sase/memory/sase_flags.md` scopes flags
  to behavior that is unproven, early-landed, or a deprecation whose old branch must
  stay reachable, and reserves agent-created `beta` flags for epic plans. This lands
  complete in one change with no branch to keep alive. A `word_case` knob next to
  `word_ranking` was considered and rejected: the old behavior is not something a user
  would choose forever, so it would be config surface with no constituency.
- **No move to the Rust core.** Prompt-history word completion is TUI-only Python today;
  `sase-core`'s `editor/completion.rs` serves the LSP surface and has no history-word or
  prompt-word code at all (verified by search). This change stays beside the existing
  Python subsystem. If the editor surface ever grows history-word completion, the casing
  policy should move to `sase-core` then — that is a separate decision, not this tale.
- **No change to matching.** Prefix matching is already case-insensitive; nothing here
  widens or narrows which words match.
- Other completion kinds (`file`, `xprompt`, `directive`, `placeholder`, `artifact_ref`,
  `jinja`) keep their current casing behavior.
- No `prompt_word_deletions.json` migration; folded comparison is backward compatible
  with entries written by the old code.

## Risks and mitigations

- **`PromptWordIndex.words` changes meaning** (exact spellings -> canonical spellings).
  Contained: the only non-test readers are `prompt_word_ranking.py` and
  `_startup_history_words.py`, both listed above. Tests reference
  `index.words.index(...)` and need the updates called out in Testing.
- **Index build cost.** Budget the post-pass at well under 10 ms on a 10k-distinct-word
  corpus (measured ~5 ms); if a `-m slow` bench is convenient, record the before/after
  build time in the change description. `sase/memory/tui_perf.md` rule 9 applies — this
  must not migrate any work onto the startup path.
- **Broadened deletions.** Deleting one row now suppresses every case variant. This is
  intended (D4) but is user-visible; mention it in the commit message.

## Verification

- `just check` after implementation; `just check-full` through `/sase_monitor` before
  landing, since this touches the shared prompt-completion widget path.
- Read `sase/memory/symvision.md` with `/sase_memory_read` if the new public helper
  trips an unused-symbol or private-misuse lint.
- Manual smoke in `sase ace`: type `SPECTAC` and press `<ctrl+t>`.
