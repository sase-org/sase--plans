---
tier: epic
status: done
title: Rank saved placeholder tags by relation, recency, and frequency
goal: "The `<` completion menu ranks saved placeholder tags by how strongly they relate
  to the prompt being written, how recently they were used, and how often they were
  used, and every saved row shows the same compact, colored signal the history-word menu
  already uses to explain its ordering.

  "
phases:
  - id: signals_core
    title: Shared ranking-signal rendering
    depends_on: []
    size: small
    description: "signals_core: extract the score meter, dominant-reason chip, age
      formatter, palette, and colored legend out of the history-word row module into a
      provider-neutral rendering module behind a structural signal protocol, leaving
      history-word output byte-identical.

      "
  - id: store
    title: Placeholder context store
    depends_on: []
    size: medium
    description: "store: grow the durable common-placeholder store to version 2 with
      per-entry context bags and corpus statistics, record that evidence on the submit
      and launch paths, read version 1 stores forward-compatibly, and backfill context
      once from prompt history.

      "
  - id: ranking
    title: Relation, recency, and frequency scoring
    depends_on:
      - store
    size: medium
    description: "ranking: add the pure scoring engine that turns the placeholder
      context store plus the prompt being edited into ranked placeholders with
      per-signal contributions, a dominant reason, and the evidence each row displays.

      "
  - id: wiring
    title: Warm cache, menu, and settings wiring
    depends_on:
      - ranking
    size: medium
    description: "wiring: hold the placeholder index in the app-global warm cache, feed
      ranked saved tags through the Rust completion engine and reattach their ranking
      evidence, keep Ctrl+D deletion instant, and add the two ranking settings.

      "
  - id: signals
    title: Ranking signals in the placeholder panel
    depends_on:
      - signals_core
      - wiring
    size: medium
    description:
      "signals: render the score meter and dominant-reason chip on saved placeholder
      rows, align both source groups on one label column, add the combined
      border-subtitle legend with an explicit width ladder, and refresh the docs and PNG
      goldens."
proposed_by: bbugyi200.athena.04e
bead_id: sase-o8
create_time: 2026-09-09 19:51:02
---

- **PROMPT:**
  [prompts/202608/placeholder_completion_ranking.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/placeholder_completion_ranking.md)
- **BEAD:**
  [sase-o8](https://github.com/sase-org/sase--beads/blob/main/pages/sase-o8/README.md)

# Plan: Rank saved placeholder tags by relation, recency, and frequency

## Problem

The `<` menu has two candidate groups. The prompt-local group (cyan `<>` badge) is
ordered by the document itself and is fine. The saved group (gold `◆` badge) is drawn
from `~/.sase/prompt_placeholders.json` and is ordered by `_display_order()`
(`src/sase/history/prompt_placeholders.py`): `count` desc, then `last_used` desc, then
`text` asc. `build_placeholder_completion_candidates()`
(`../sase-core/crates/sase_core/src/editor/placeholder.rs`) preserves that caller order
verbatim after its prefix filter and dedup, so the persisted order _is_ the menu order.

That lexicographic ordering has the same practical failures the history-word menu had
before `sase-na`:

1. `count` dominates absolutely. A tag written twelve times a year ago outranks a tag
   written four times this morning, because `last_used` is only a tiebreak within an
   identical count.
2. Nothing in the ordering knows what the current prompt is about, even though the
   prompt is sitting right there in the widget — and placeholders are exactly the kind
   of token that travels in packs (`<epic name>` with `<phase title>`, `<file>` with
   `<line>`).
3. The panel shows at most 8 content rows, and the saved group sits _below_ the whole
   prompt-local group, so in practice a bad first few saved rows is the entire
   saved-group experience.

The rows also carry no explanation, so a user cannot tell whether a saved row is on top
because it is fresh, because it is common, or because it fits the prompt — while the
history-word menu one keystroke away answers exactly that question with a meter, a chip,
and a legend. This epic closes that gap.

## Design

### What changes and what does not

Only the **saved** group is reordered. Prompt-local ordering (live spans in document
order, then literal-zone spans in document order) is a property of the document and
stays exactly as it is; it is not a ranking problem, and reordering it would mean
changing Rust core for no user benefit. Prompt-local rows therefore carry no ranking
evidence and render exactly as they do today.

### Ranking model

Every saved placeholder gets three normalized signals in `[0, 1]` and a weighted
composite, deliberately the same policy the history-word menu uses so the two menus read
alike:

```
score = 0.50 * relation + 0.30 * recency + 0.20 * frequency
```

**Frequency** saturates, so one runaway tag cannot flatten the scale:

```
frequency(p) = log1p(count(p)) / log1p(max_count)
```

`count(p)` is already persisted and already means "number of prompts that used this tag"
(a tag repeated inside one prompt counts once — `record_prompt_placeholders()` dedups by
distinct inner text).

**Recency** is an exponential decay of the tag's newest use:

```
recency(p) = 0.5 ** (age_seconds(p) / RECENCY_HALF_LIFE_SECONDS)   # 14 days
```

The half-life is 14 days, not the history-word engine's 7. The two corpora are not
comparable: the word index holds ~10,000 words drawn from thousands of prompts, while
this store holds at most `common_placeholder_count` (default 100) entries under LRU
retention, so its median entry is weeks old. At a 7-day half-life nearly every row's
recency term would round to zero and frequency would silently become the only live
signal. Age is computed against a `now` captured once per ranking call, never baked into
the store, so a long-lived ACE session cannot drift into stale recency.

**Relation** answers "do prompts like this one use this tag?" It needs evidence the
store does not have today, so the store grows one: each entry keeps a bounded bag of
**context tokens** observed in the prompts that used it, and the store keeps the corpus
statistics needed to normalize them.

A prompt's context tokens are drawn from one namespace with two flavors:

- **prose tokens** — `_PROMPT_WORD_RE` matches over the prompt text, casefolded, at
  least `CONTEXT_MIN_WORD_LENGTH` (4) characters.
- **tag tokens** — the literal `<text>` spelling of every distinct placeholder in that
  prompt, so tag-to-tag co-occurrence is a first-class signal rather than a special
  case.

Given the current prompt's context token set `C`, for candidate `p`:

```
idf(c)        = log(prompt_count / df(c))                              # skipped if <= 0
observed(p)   = Σ_{c∈C} idf(c) * context_count(p, c) / context_uses(p)
background(p) = Σ_{c∈C} idf(c) * df(c) / prompt_count
lift(p)       = observed(p) / background(p)
hits(p)       = |{c ∈ C : context_count(p, c) > 0}|
relation(p)   = min(1, log2(lift) / log2(8)) * hits / (hits + 2)       # 0 when lift <= 1
```

`background(p)` is the same weighted sum under the null hypothesis that `p` is
independent of the context, so an unrelated tag scores `lift ≈ 1` and therefore
`relation = 0` no matter how common it is — the frequency term already covers "common".
`log2(lift)/log2(8)` means "8× more likely in prompts like this one than in an average
prompt" saturates the signal, and `hits/(hits + 2)` keeps a tag that matched exactly one
context token from claiming a perfect score. Relation is forced to `0` for every
candidate when the store has recorded fewer than `RELATION_MIN_PROMPTS` (8) prompts,
where the statistics are meaningless.

This is the same capped-lift-times-shrinkage shape as `prompt_word_ranking.py`, computed
from per-entry bags instead of prompt postings. The bags are not aged: a 100-entry LRU
store whose bags hold at most 24 tokens is already recency-bounded by eviction, so a
second decay term would be tuning noise. Sorting is `(-score, text.casefold(), text)`,
so ties are deterministic and tests are stable.

#### Alternatives considered

- **Derive relation from the warm `PromptWordIndex`** by scoring a tag's own words
  (`<phase title>` → `phase`, `title`) against the already-computed context. It needs no
  new persisted data, but it measures whether the tag's _wording_ fits the prompt, not
  whether the tag is _used_ in prompts like it — and it collapses for tags whose words
  are generic (`<the plan>`, `<name>`). Rejected as the primary signal; the store-backed
  evidence answers the question directly.
- **Co-occurrence with the current prompt's other tags only.** Strong evidence, but it
  is empty in the single most common case: reaching for the _first_ tag in a prompt.
  Folding tag tokens into the same namespace as prose tokens keeps that strength without
  the blind spot.

### Where the code lives

`sase/memory/rust_core_backend_boundary` asks for shared backend behavior to live in
`../sase-core`. **No phase of this epic changes the Rust core, and no phase needs to
open that repo.** The placeholder engine there — span extraction, cursor context, prefix
filter, dedup, replacement range — is already shared with the xprompt LSP and is
unchanged. Its contract already declares `common` as caller-ranked ("They are appended,
in the order given"), and `crates/sase_xprompt_lsp/src/server.rs` passes `&[]` with the
standing comment that the LSP has no common-placeholder source of its own. Ranking a
caller-owned durable store is a caller concern by that existing design, and the store
itself lives in the Python-only `src/sase/history/` subsystem alongside prompt history.
`prompt_word_ranking.py` set the precedent for keeping this pure and TUI-free; doing the
same here keeps a future port mechanical rather than harder.

Scoring stays a pure, dependency-light module in `src/sase/history/` with no Textual
imports.

### Not a feature flag

`sase/memory/sase_flags.md` is explicit: if users are meant to choose the value forever,
it was never a feature flag. `placeholder_ranking` and `placeholder_ranking_signals` are
permanent config fields, exactly like the `word_ranking` pair that shipped unflagged
with `sase-na`. **No phase should run `sase flag new`.**

### Visual signal

```
╭─ placeholder ──────────────────────────────────────────────────────────────╮
│ ▸ <> release note                                                          │
│   <> release owner                                                         │
│   ◆  phase title       ▰▰▰▰▱  ⇄ <epic name>                                │
│   ◆  plan file         ▰▰▰▱▱  ✦ 12×                                        │
│   ◆  worker            ▰▱▱▱▱  ◷ 3d                                         │
╰─ <> prompt  ◆ saved  ⇄ related · ◷ recent · ✦ frequent ────── [^D] delete ─╯
```

Saved rows keep their gold `◆` badge and gain the meter and chip; prompt rows keep their
cyan `<>` badge and gain nothing. Both groups align on one label column measured across
the whole visible window, badges included, so the menu stays a single table rather than
two. The meter, chip, palette, glyphs, and legend are literally the same code the
history-word menu uses — phase `signals_core` makes them shared before phase `signals`
consumes them.

### Perf

All disk work stays on the existing off-thread warm path
(`src/sase/ace/tui/actions/_startup_common_placeholders.py`); the keystroke path stays
read-only and allocation-light per `sase/memory/tui_perf.md` rules 1, 8, and 11. Per
menu open the ranker tokenizes the prompt once and builds the context set; per keystroke
it scores at most `common_placeholder_count` (100) entries against at most
`CONTEXT_MAX_TOKENS` (24) tokens — under 2,400 dict lookups plus a 100-element sort,
comfortably sub-millisecond inside the 16 ms key-to-paint budget. The context is
memoized on the index object keyed by the context token tuple, mirroring
`PromptWordIndex._ranking_memo`; because a warm-cache rebuild replaces the whole index
object, that memo can never serve results from a stale corpus.

Recording grows the submit-path write from ~4 KB to roughly 100 KB (2,000 vocabulary
entries plus 100 bags of 24). That path is already an atomic tempfile write under an
exclusive lock, runs only on prompt submit and agent launch, and is never on the
keystroke path.

### Cross-phase gotcha: symvision

`sase-na` lost a full `just check-full` cycle to this, per the notes on that bead. A
symbol introduced by one phase but not consumed until a later phase trips
`just _lint-symvision`. The introducing phase adds
`--epic-symbol "<its phase bead id>(Symbol)"` entries to the `_lint-symvision` recipe in
`Justfile`, and **the consuming phase deletes them in the same change that lands the
consumer**. Read `sase/memory/symvision.md` with `/sase_memory_read` before adding or
removing any of them.

## Shared ranking-signal rendering

Pure refactor with no behavior change. Add
`src/sase/ace/tui/widgets/_ranking_signal_rows.py` and move into it, unchanged, from
`src/sase/ace/tui/widgets/_history_word_rows.py`:

- the palette and glyph constants `RELATION_COLOR`/`RECENCY_COLOR`/`FREQUENCY_COLOR` and
  `RELATION_GLYPH`/`RECENCY_GLYPH`/`FREQUENCY_GLYPH`, plus
  `_REASON_GLYPHS`/`_REASON_COLORS`;
- `_build_score_meter()` and `_meter_cell_colors()` (5 cells, largest-remainder
  distribution in a fixed relation → recency → frequency order);
- `_format_reason_chip()` and `_format_age()`;
- the legend builder currently inlined in `history_word_completion_subtitle()`
  (`_prompt_input_bar_completion_panel_labels.py`), exposed as
  `ranking_signal_legend() -> Text`.

The renderers must not depend on `HistoryWordCompletionMetadata`. Define instead:

```python
@runtime_checkable
class RankingSignals(Protocol):
    reason: str
    related_to: str
    use_count: int
    age_seconds: float
    score: float
    relation: float
    recency: float
    frequency: float
```

`HistoryWordCompletionMetadata` already declares exactly those eight fields and
satisfies the protocol structurally, so **that dataclass is not modified**. Also lift
the label-column measurement into `ranking_label_width(display, *, badge_cells, cap)`;
`history_word_label_width()` becomes a thin wrapper passing `badge_cells=0` and
`cap=_LABEL_WIDTH_CAP` (28).

`_history_word_rows.py` keeps `append_history_word_completion_row()` and its
`HistoryWordCompletionPlaceholder` handling and imports the rest.
`_prompt_input_bar_completion_panel_labels.py` imports the palette from the new module
rather than from `_history_word_rows`.

**Tests**: `tests/ace/tui/widgets/test_history_word_rows.py` is the contract — it must
pass with only import-path edits, and no assertion may be weakened. Split the
meter/chip/age/legend cases that are now provider-neutral into
`tests/ace/tui/widgets/test_ranking_signal_rows.py`, exercising them through the shared
API with a small test-local dataclass satisfying `RankingSignals` (proving the protocol
is genuinely provider-neutral). `history_word_completion_panel_120x40.png` must not
change; a regenerated golden here means the refactor was not behavior-preserving.

## Placeholder context store

All in `src/sase/history/prompt_placeholders.py`.

**Format version 2.** `_STORE_VERSION` becomes `2`:

```json
{
  "version": 2,
  "prompt_count": 412,
  "context_frequency": { "epic": 88, "phase": 60, "<phase title>": 12 },
  "placeholders": [
    {
      "text": "epic name",
      "count": 12,
      "last_used": "260817_101500",
      "context_uses": 12,
      "context": { "epic": 9, "phase": 7, "<phase title>": 5 }
    }
  ]
}
```

- `prompt_count` — recorded prompts that contributed context; the denominator for `df`.
- `context_frequency` — token → number of recorded prompts containing it.
- `context` — token → number of _this entry's_ context-contributing prompts containing
  it.
- `context_uses` — how many prompts contributed to this entry's bag. It is tracked
  separately from `count` so `context_count / context_uses` is always a true share in
  `[0, 1]`; the history backfill below can otherwise contribute more context prompts
  than the store has recorded uses.

Entries stay persisted in `_display_order()` (`count` desc, `last_used` desc, `text`
asc), so reads remain a plain sequential parse and the `recent` ranking mode is
byte-identical to today's menu.

**Constants**: `CONTEXT_MIN_WORD_LENGTH = 4`, `CONTEXT_TOKENS_PER_PROMPT = 24`,
`CONTEXT_STOPWORD_RATIO = 0.20`, `CONTEXT_TOKENS_PER_ENTRY = 24`,
`CONTEXT_VOCABULARY_LIMIT = 2000`. `CONTEXT_MIN_WORD_LENGTH` is deliberately independent
of `ace.prompt_completion.word_min_length` (5): that setting governs which words are
offered as completions, while this governs which words are usable evidence, and `epic`,
`plan`, `bead` must qualify.

**`_prompt_context_tokens(text, *, context_frequency, prompt_count)`** returns the
prompt's bounded context tokens: tag tokens for every distinct placeholder (`raw` and
literal-zone alike — a tag written in a code fence is still evidence about the prompt),
plus prose tokens from `_PROMPT_WORD_RE` (imported from `prompt_word_index`, not
re-implemented), casefolded and length-filtered. Drop prose tokens whose
`df / prompt_count` exceeds `CONTEXT_STOPWORD_RATIO`, then keep at most
`CONTEXT_TOKENS_PER_PROMPT`: all tag tokens first (few, and the strongest evidence),
then the rarest prose tokens, ties broken by token text so truncation is deterministic.

**Recording.** `record_prompt_placeholders(text)` keeps its exact contract —
best-effort, never raises, never blocks submit, `limit <= 0` disables it — and
additionally, inside the existing lock and read/modify/write cycle: computes the
prompt's context tokens once, increments `prompt_count` and `context_frequency`, and for
each recorded tag increments `context_uses` and its bag over the context tokens
**excluding its own `<text>` token** (a tag always co-occurs with itself; that token
would be a constant carrying no information). Bags are trimmed to
`CONTEXT_TOKENS_PER_ENTRY` by lowest count then largest token, and `context_frequency`
to `CONTEXT_VOCABULARY_LIMIT` by lowest `df` then largest token; a token evicted from
the vocabulary but still present in a bag simply contributes nothing to relation (phase
`ranking` skips tokens with no `df`). Both trims are logged at debug level when they
bite, per the project's no-silent-caps habit.

The two call sites in `prompt_store_mutations.py` and
`src/sase/ace/tui/actions/agent_workflow/_launch_start.py` pass the same prompt text
they pass today; neither changes.

**Reading.** `_entries_from_data()` accepts `version` 1 **and** 2. A version-1 payload
yields today's entries with empty bags, `context_uses = 0`, `prompt_count = 0`, and an
empty vocabulary — never a discarded store. This is the trap this phase exists to avoid:
bumping `_STORE_VERSION` without a forward-compatible reader would make every existing
store parse as empty, and because `seed_common_placeholders_from_history()` only fires
when the file is _absent_, nothing would ever refill it. Missing, malformed, or negative
statistics degrade to zero rather than raising, exactly like the existing per-field
validation.

Add `load_common_placeholder_index() -> CommonPlaceholderIndex`, a frozen slots
dataclass holding the ordered entries plus `prompt_count`, `context_frequency`,
`max_count`, and one private mutable memo slot for phase `ranking`. Keep
`load_common_placeholders(limit)` as a thin wrapper returning display-order texts, so
today's callers and tests are untouched. Keep `common_placeholder_source_token()` and
`remove_common_placeholder(text)` with their current signatures; removal drops the entry
and leaves `prompt_count`/`context_frequency` alone, because deleting one saved tag does
not un-observe the prompts the corpus was measured over.

**Seeding and backfill.** `seed_common_placeholders_from_history(limit)` runs when the
store is absent **or** when it parses at version 1, and its bounded `_SEED_SHARD_LIMIT`
(24) scan now also accumulates context bags, `context_frequency`, and `prompt_count`. On
the version-1 upgrade path it **merges rather than replaces**: existing `count` and
`last_used` are authoritative and are preserved exactly; only context evidence and
corpus statistics come from the scan; texts the scan finds that the store lacks are
added as ordinary seed entries. The existing "another process wrote the store while this
scan ran" re-check still applies, and on the upgrade path it must re-read and re-merge
rather than bail, so a submit racing the upgrade cannot lose its recorded use. Return
`True` when a store was written.

**Tests** (`tests/history/test_prompt_placeholders.py`, whose existing cases must keep
passing unchanged — that is this phase's no-regression contract):

- A version-1 store on disk loads with its counts and `last_used` intact, is upgraded in
  place by the backfill, and no entry is lost; a version-1 store that is _never_
  upgraded still serves correct `recent`-order texts.
- Recording increments `prompt_count`, `context_frequency`, `context_uses`, and bags; a
  tag repeated in one prompt still counts once and its bag still moves once.
- A tag's own `<text>` token never appears in its own bag; another tag in the same
  prompt does.
- Tag tokens survive the stopword filter and the per-prompt cap even when many prose
  tokens compete; prose tokens above `CONTEXT_STOPWORD_RATIO` are dropped.
- Per-entry and vocabulary trims are deterministic under ties and preserve the
  highest-count tokens.
- LRU retention still evicts by `last_used` and still persists `_display_order()`; an
  evicted entry's bag goes with it while `prompt_count` and `context_frequency` remain.
- `remove_common_placeholder()` leaves corpus statistics intact.
- Corrupt, truncated, empty, and unknown-version files still read as an empty store, and
  a failed write still leaves the previous store readable.
- `common_placeholder_count: 0` still disables recording, loading, and seeding.
- Recording is still best-effort: a raising `_save_payload` is swallowed and the
  caller's prompt still submits.

## Relation, recency, and frequency scoring

Add `src/sase/history/prompt_placeholder_ranking.py`, importing only the store module
and the standard library — no Textual, no widget imports.

**Constants** (one documented block; this is the tuning surface of the feature):
`RELATION_WEIGHT = 0.50`, `RECENCY_WEIGHT = 0.30`, `FREQUENCY_WEIGHT = 0.20`,
`RECENCY_HALF_LIFE_SECONDS = 14 * 86400`, `RELATION_LIFT_CAP = 8.0`,
`RELATION_SHRINKAGE = 2.0`, `RELATION_MIN_PROMPTS = 8`, `CONTEXT_MAX_TOKENS = 24`.

**`RankedPlaceholder`** (frozen slots dataclass) carries `text`, `score`, the three
_weighted_ contributions `relation`/`recency`/`frequency` (so the meter needs no
re-derivation), `reason: Literal["relation", "recency", "frequency"]`,
`related_to: str`, `use_count: int`, and `age_seconds: float`. Its field names and
semantics match `RankedWord` so the shared `RankingSignals` renderer consumes both.
`reason` is the argmax of the weighted contributions with a fixed relation → recency →
frequency tiebreak, and is `"recency"` when every contribution is zero. `related_to` is
the context token with the largest `idf(c) * context_count(p, c) / context_uses(p)`
contribution, kept in its stored spelling so a tag token displays as `<phase title>` and
a prose token as `phase`.

**`build_placeholder_ranking_context(index, text, *, now)`** tokenizes the prompt with
the same `_prompt_context_tokens()` rule the store records with — one implementation, so
evidence and queries can never drift — caps the result at `CONTEXT_MAX_TOKENS`, and
returns a context object holding the token tuple and each token's `idf`. It is memoized
in the index's private mutable slot keyed by the token tuple. Note there is no
cursor-word exclusion to make here: the tag being typed is an incomplete `<prefix` with
no closing bracket, so `placeholder_spans()` never reports it, and its prose characters
are a strictly weaker signal than the surrounding prompt.

**`rank_common_placeholders(index, context, *, now)`** scores every entry, sorts by
`(-score, text.casefold(), text)`, and returns `list[RankedPlaceholder]`. It ranks the
whole store rather than a prefix slice: the store is capped at 100 entries, the Rust
engine owns prefix filtering and dedup against the prompt group, and pre-filtering here
would duplicate that rule in a second place. `relation` is zero for every entry when
`index.prompt_count < RELATION_MIN_PROMPTS` or the context is empty.

**`rank_recent_common_placeholders(index, *, now)`** is the `recent` mode path: stored
display order, zeroed contributions, no reason chip, so downstream code has exactly one
row type.

**Timestamps**: `last_used` is a `%y%m%d_%H%M%S` local-time SASE stamp. Parse it by
fixed-width integer slicing plus `time.mktime`, matching `prompt_word_index`; an
unparsable or empty stamp yields epoch `0.0` — very old, never a crash. Ages clamp at
`0.0`, so a clock skew into the future yields `recency == 1.0` rather than a value above
one.

**Tests** (`tests/history/test_prompt_placeholder_ranking.py`) build small in-memory
stores through the phase `store` API and assert behavior, not float literals:

- A tag whose bag matches the prompt's context outranks a more recent, more frequent
  unrelated tag; with the context words removed from the prompt, the order flips back to
  recency and frequency.
- A tag co-occurring with the prompt's _other placeholder_ is lifted with
  `reason == "relation"` and a `related_to` that renders with brackets.
- A tag present in nearly every prompt (`lift ≈ 1`) gets `relation == 0` and does not
  displace a genuinely associated tag.
- Shrinkage: one matched context token scores strictly below the same lift with many
  matches.
- Recency halves at the half-life; a future stamp clamps to `1.0`; an unparsable stamp
  ranks last rather than raising.
- Frequency saturates and cannot on its own outrank a strong relation-plus-recency pair.
- Stores below `RELATION_MIN_PROMPTS`, stores with an empty vocabulary, and prompts with
  no usable context all produce zero relation everywhere and never raise.
- A context token missing from `context_frequency` (evicted from the vocabulary)
  contributes nothing and does not divide by zero.
- Determinism: equal scores sort by folded text then text; identical inputs give
  identical order across runs.
- The memo returns the identical context object for the same token tuple and recomputes
  for a different one.
- `recent` mode reproduces `load_common_placeholders()` order exactly and attaches no
  evidence.

## Warm cache, menu, and settings wiring

**Settings** — add to `PromptCompletionSettings`
(`src/sase/ace/tui/widgets/prompt_completion.py`) and
`parse_prompt_completion_settings()`, mirroring the `word_ranking` pair including its
`_parse_word_ranking_mode()`-style normalization of unknown values:

- `placeholder_ranking: Literal["smart", "recent"] = "smart"` — `smart` is the new
  ranking, `recent` restores today's exact stored order and suppresses the signal
  column.
- `placeholder_ranking_signals: bool = True` — render the meter, chip, and legend.

Mirror both keys in `src/sase/default_config.yml` under `ace.prompt_completion` and in
`src/sase/config/sase.schema.json` with descriptions and defaults.

**Warm cache** (`src/sase/ace/tui/actions/_startup_common_placeholders.py`) — the cached
payload becomes a `CommonPlaceholderIndex | None` instead of `list[str] | None`:

- `common_placeholders()` keeps its name and its `list[str] | None` return, derived once
  from the warm index and cached alongside it, so existing callers and lightweight test
  harnesses keep working; add `common_placeholder_index()` returning the index.
- The off-thread load, the `previous_token` staleness short-circuit, the
  seed-on-first-warm call, the `limit <= 0` disabled path, the in-flight/pending
  coalescing guards, and `common_placeholders_generation()` all keep their current shape
  and semantics.
- `forget_common_placeholder(text)` drops the entry from the warm index (leaving
  `prompt_count` and `context_frequency` untouched, matching the store) instead of
  filtering a string list, so `Ctrl+D` stays instant and still forces a store-backed
  next warm.

**Menu** (`src/sase/ace/tui/widgets/placeholder_completion.py`) — extend the row
metadata without disturbing the source contract:

```python
@dataclass(frozen=True, slots=True)
class PlaceholderCompletionMetadata:
    source: PlaceholderCandidateSource
    ranking: PlaceholderRankingMetadata | None = None
```

`PlaceholderRankingMetadata` declares the same eight fields as
`HistoryWordCompletionMetadata` and therefore satisfies phase `signals_core`'s
`RankingSignals` protocol. `source` keeps its position and default-free semantics, so
`_placeholder_candidate_source()`, the `Ctrl+D` guard in `_file_completion_accept.py`,
and the existing PNG fixture keep working unchanged.

Add
`build_indexed_placeholder_completion_result(text, cursor_offset, index, *, now, smart, include_common_when_prefix_empty)`:

1. rank the store (`rank_common_placeholders` or `rank_recent_common_placeholders`);
2. pass the ranked texts to `placeholder_completion()` as `common` — Rust preserves
   caller order and owns prefix filtering, dedup against the prompt group, and the
   replacement range, exactly as today;
3. map each returned `common` candidate back to its `RankedPlaceholder` by exact text
   (Rust dedups `common` by exact string, so the mapping is total and unambiguous) and
   attach `PlaceholderRankingMetadata`; attach nothing in `recent` mode or to `prompt`
   candidates;
4. apply the existing empty-prefix `drop_common` rule and return the same
   `PlaceholderCompletionResult` shape.

Keep the existing `Sequence[str]` overload for callers and tests without an index.
`build_placeholder_completion_result()`'s current behavior with an explicit `common`
list must be unchanged — the existing tests in
`tests/ace/tui/widgets/test_placeholder_completion.py` are that contract.

`_placeholder_completion_at_cursor()` (`_placeholder_highlight.py`) routes through the
new builder when the app exposes an index, passing `now` explicitly so tests can pin it.
Its memo key already folds in `common_placeholders_generation()` and its slot map is
already cleared on every text change, which is exactly what the context-dependent result
needs; no key change is required, and the docstring should say why.
`placeholder_lone_leading_match()` is unaffected: it groups by source, not by order
within a group.

**Tests** — extend `tests/ace/tui/widgets/test_placeholder_completion.py` (which also
owns the `Ctrl+D` delete cases) and `tests/ace/tui/test_common_placeholders_cache.py`;
split the file if the additions push it past the `_lint-toobig` limit:

- End-to-end ranked ordering from a seeded store, including a case where the top-ranked
  saved tag is neither the most frequent nor the most recent.
- Prompt-local candidates keep document order and precede every saved candidate
  regardless of score; a saved tag that also appears in the prompt is still emitted
  once, as a prompt candidate.
- `placeholder_ranking: recent` reproduces today's order exactly and attaches no
  metadata.
- The bare-`<` rule is unchanged: auto trigger hides the saved group, `Ctrl+T` shows it
  ranked.
- `common_placeholder_count: 0` still disables the saved group end to end.
- `Ctrl+D` still refuses prompt rows, still removes a saved row instantly, still
  notifies, and no longer needs a rebuilt store to take effect.
- A cold cache renders the prompt group alone and the menu re-ranks once the cache
  publishes.
- Settings parsing: defaults, `recent`, unknown values falling back to `smart`, and
  non-boolean `placeholder_ranking_signals`.

## Ranking signals in the placeholder panel

**Rows.** Add `src/sase/ace/tui/widgets/_placeholder_rows.py` (or extend
`_prompt_input_bar_completion_rows_simple.py`'s placeholder renderer in place if it
stays under the `_lint-toobig` limit) so `append_placeholder_completion_row()` takes
`label_width`, `inner_width`, and `signals_enabled`, and:

- renders today's badge and label for every row;
- for saved rows carrying `PlaceholderRankingMetadata`, appends the shared meter and
  chip from `_ranking_signal_rows`;
- degrades by width, never by clipping — the chip is dropped first, then the meter,
  leaving exactly today's row;
- renders today's row unchanged for prompt rows, for `placeholder_ranking: recent`, and
  when `placeholder_ranking_signals` is off.

Add `placeholder_label_width(candidate)` on top of the shared
`ranking_label_width(display, *, badge_cells, cap)` and join it to `_RowLayout` in
`_prompt_input_bar_completion_panel_content.py`, measured across **both** source groups
with their badge cells included, so prompt and saved rows share one column.

**Subtitle.** `placeholder_completion_subtitle(visible, inner_width)` in
`_prompt_input_bar_completion_panel_labels.py` composes, in one monotone width ladder:

1. `<> prompt  ◆ saved` + `⇄ related · ◷ recent · ✦ frequent` + `[^D] delete`;
2. drop the source legend (the badges are visible in the rows; the meter colors are
   not);
3. drop the signal legend, leaving today's subtitle exactly;
4. `[^D] delete` alone.

The source legend still only appears when both groups are visible, and the signal legend
only when some visible row carries ranking metadata. Branch to it in
`show_file_completions()` (`_prompt_input_bar_completion_panel.py`) under
`kinds.placeholder and placeholder_ranking_signals`, mirroring the existing
`kinds.history_word` branch, and thread `placeholder_ranking_signals` from
`_file_completion_base.py` alongside the existing `word_ranking_signals` argument.
`completion_delete_subtitle()`'s other providers are untouched.

**Docs.** Update the Placeholder-completion bullet in the `docs/ace.md` Completion
section: replace "ranked by use count and recency" with the three signals and their
weights, describe the meter, chip, and legend and their width degradation, and document
`placeholder_ranking`/`placeholder_ranking_signals`. The `?` help modal and the
keybinding table need no change — no key or key semantics change here.

**Tests**:

- Row unit tests: meter and chip present on saved rows, absent on prompt rows, absent
  under `recent` and under `placeholder_ranking_signals: false`; the two-step width
  degradation; both groups aligned on one column when badge widths differ.
- Subtitle unit tests for all four ladder rungs, for "no visible saved rows", and for
  "no row carries metadata".
- Widget tests rendering through `show_file_completions()` asserting row text and border
  subtitle at a wide and a narrow panel.
- Refresh both PNG goldens
  (`tests/ace/tui/visual/test_ace_png_snapshots_placeholder_completion.py` →
  `placeholder_completion_panel_120x40.png` and
  `placeholder_common_completion_panel_120x40.png`) by extending the fixtures to attach
  ranking metadata to saved candidates and regenerating with
  `--sase-update-visual-snapshots`. The goldens are the acceptance evidence that the
  panel actually looks right; the visual suite's glyph audit already covers `▰`, `▱`,
  `⇄`, `◷`, and `✦` from `sase-na`.

## Verification

Every phase leaves the tree green with `just check` and must run `just install` first,
since workspace virtualenvs go stale. Phases `signals_core` and `signals` additionally
run `just test-visual`. The combined tree is landed behind `just check-full` through
`/sase_monitor` with a `--next` action, never inline.
