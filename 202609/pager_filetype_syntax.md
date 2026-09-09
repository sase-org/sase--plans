---
tier: epic
status: done
title: File-aware syntax highlighting for the SASE pager
goal:
  Make source files, Markdown documents, and diffs quietly beautiful in every pager
  entry point while preserving source text, existing styles, link actions, search, and
  responsiveness.
phases:
  - id: language_contract
    title: Shared source-language policy and Python binding
    depends_on: []
    size: medium
    description:
      "language_contract: implement deterministic language selection, filename
      provenance, the additive Rust wire/API, and its thin Python facade with binding
      coverage. Do not activate rendering."
  - id: syntax_engine
    title: Offset-preserving syntax spans and adaptive palette
    depends_on: []
    size: medium
    description:
      "syntax_engine: implement bounded Pygments token spans, corrected Markdown
      frontmatter and fence offsets, source-style preservation, and a restrained
      theme-adaptive palette without activating a production call site."
  - id: reading_surface
    title: Responsive syntax composition and styled search
    depends_on:
      - language_contract
      - syntax_engine
    size: medium
    description:
      "reading_surface: implement optional section metadata, pump-free preparation,
      bounded caches, source-preserving label composition, styled search, theme
      invalidation, and the language hint. Exercise explicit test inputs while leaving
      existing producers inert."
  - id: activate_and_verify
    title: Enable all pager entry points and verify the finished experience
    depends_on:
      - language_contract
      - syntax_engine
      - reading_surface
    size: medium
    description:
      "activate_and_verify: propagate real source provenance, support recognized source
      paths, add syntax controls and config, honor color policy, document behavior, and
      complete functional, PNG, and responsiveness verification."
proposed_by: bbugyi200.athena.03g
bead_id: sase-xz
create_time: 2026-09-09 19:52:31
---

- **PROMPT:**
  [prompts/202609/pager_filetype_syntax.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/pager_filetype_syntax.md)
- **BEAD:**
  [sase-xz](https://github.com/sase-org/sase--beads/blob/main/pages/sase-xz/README.md)

# File-aware syntax highlighting for the SASE pager

## Outcome and scope

Opening `src/sase/cli_pager.py` should distinguish comments, strings, and language
structure immediately after the first paint. A research document should show YAML
frontmatter, readable Markdown source, and syntax inside labeled code fences. A diff
should distinguish additions, deletions, and hunk headings. Links remain the clearest
interactive objects, and entering `/` retains the document's colors beneath search
matches. None of these operations changes the underlying document characters.

This is an epic because the shared Rust policy/binding, Python lexer/palette, and
Textual lifecycle/search work have distinct contracts and can be implemented by separate
bounded workers. The first two phases are independent; all dependencies are declared
above. Each implementation phase is medium, following `sase_sizes.md`.

Only planning is authorized in the proposing turn. Implementation starts after the plan
handoff and approval. This document is a scratch plan, not a source change.

## Evidence and design decisions

The primary design input was read with `sase artifact read`:
`research:202609/pager_filetype_syntax_layer/pager_filetype_syntax_layer.md`. Relevant
project guidance was read through `sase memory read`: `sase_sizes.md`,
`sase_artifacts.md`, `tui_perf.md`, `cli_rules.md`, `sase_flags.md`, and
`lint_and_test.md`. Implementation workers must reread applicable guidance in their own
run and follow each repository's instructions.

Current code establishes the integration seams:

- `pager/document.py` derives a canonical `Text` once; `plain_text` and attached target
  offsets come from it. `body_text` returns a copy.
- `pager/_layout.py` measures and paints sections through `_section_renderable`.
  `pager/_labels.py` currently starts from `section.body_text`, so simply coloring an
  unrelated renderable would lose colors whenever labels are present.
- `pager/_screen_body.py` caches by width and composes synchronously on mount. Lexing
  there would delay first paint. Label-window and prefix changes can also invalidate
  composition; they must never become lexer triggers.
- `pager/_screen_search.py` hosts the shared `VimSearchController`, whose current
  overlay starts from unstyled `Text(self.corpus)`.
- `pager/adapters.py`, `pager/resolve.py`, `artifact_cli/read.py`, `cli_pager.py`, and
  `ace/tui/actions/hints/_files.py` supply the real entry points. Artifact resolution
  currently loses original source-name information at some seams.
- `pager/resolve.py` uses a limited MIME/suffix text gate. A correct language detector
  alone does not make extensionless scripts and all supported source files reachable
  through `sase pager`.
- `main/parser_pager.py` already parses `--color` and `--wrap`; the handler does not
  apply them. This feature wires color policy; wrap behavior is outside scope.

Read-only local probes reproduced Rich `Syntax.highlight()` adding a newline to empty
input, expanding tabs, and normalizing CRLF. Pygments `get_tokens_unprocessed()`
preserved original offsets for the simple Python and non-fenced Markdown probes,
including Unicode. A second probe found a critical exception to the research's claim:
Markdown's nested lexer returns fence-body-local offsets for labeled code blocks. The
installed `_handle_codeblock` implementation even notes this offset defect. The existing
`FrontmatterMarkdownLexer` propagates it; it also misses CRLF frontmatter and a closing
YAML fence at EOF. Filename probes confirmed missing mappings for TCSS, Justfile,
`uv.lock`, and extensionless README. These are API probes, not implementation tests or
performance measurements against the project's pinned development environment.

Adopt the research's token-span approach, conservative producer-style protection,
bounded work, and restrained palette. Strengthen or adjust these recommendations:

1. Shared language policy belongs in Rust now, as required by the current backend
   boundary. Python owns Pygments execution and Rich/Textual presentation.
2. Prepare syntax after first paint in a pump-free worker, even below the caps. The
   existing synchronous layout cost does not justify adding synchronous lexing.
3. Preserve styles during search in this delivery. Color disappearing on `/` would make
   the normal reading experience feel unfinished.
4. Derive colors from the actual host theme. Preserve the standalone app's current
   default theme and ACE's chosen theme; do not recolor every existing pager just to
   accommodate the new layer.
5. Enforce legibility and hierarchy without a universal upper contrast bound tied to
   every link hue. Such a bound may be impossible in light/custom themes.
6. Handle embedded code as explicit regions with corrected absolute offsets. Raw
   tokenization alone is not sufficient for Pygments' composite lexers.
7. Use verified language mappings. Do not call a Justfile a Makefile, or turn
   `README.rst` into Markdown via a broad `README*` override.

## Product contract

### Detection and controls

Resolve each section independently; document origin, section kind, and display title are
insufficient evidence on their own. Keep target identity and provenance distinct from
syntax identity.

Selection precedence for an eligible raw source is:

1. Explicit `-s/--syntax ALIAS` selects a supported Pygments alias; `-s none` disables
   the added layer and `-s auto` requests normal detection.
2. Trusted adapter provenance identifies an actual Markdown document or diff body. A
   bead/card containing a reference to Markdown is still a formatted card.
3. The shared Rust filename policy identifies a supported source language.
4. For extensionless raw files only, inspect a bounded first-line shebang.
5. For otherwise untyped stdin only, recognize a convincing unified/git diff in a
   bounded prefix; otherwise use plain text.

The override applies to eligible initial sections, including stdin and multi-file
inputs. Newly followed targets use their own detection; revisiting history restores the
original section override. `none`, config disabling, and color disabling are session
controls and remain effective across follow/back/forward operations.

Do not guess arbitrary content with Pygments, infer a file type from `--title`, or open
paths mentioned in text to detect a language. Invalid explicit aliases fail with exit
code 2 and a concise error before consuming stdin or launching Textual. `text`/`plain`
lexer aliases produce plain text without a language chip.

Add permanent configuration `pager.syntax: auto | never`, default `auto`, using the
normal merged config/cache path. CLI `--syntax` overrides this default. `--color never`
always wins and disables added syntax; use the standalone terminal color policy to
suppress color throughout its pager UI while preserving non-color affordances. Respect
the established `NO_COLOR` behavior in auto mode; explicit `--color always` overrides
color auto-detection, but does not force an app for redirected stdout. An embedded pager
inherits its host's terminal color policy.

`--plain`, redirected output, no usable terminal, and `page_or_print`'s direct branch
retain their current output and newline rules. These paths do no lexing, syntax theme
construction, or syntax worker scheduling. `--syntax none` preserves existing producer
ANSI and link styling; it is different from `--color never`.

### Initial supported automatic types

The core owns one tested mapping table, with canonical identities independent of
Rich/Pygments objects. Cover the following common families; use ordinary Pygments
aliases for their canonical spelling where possible:

| Input                                                                   | Automatic language                                                           |
| ----------------------------------------------------------------------- | ---------------------------------------------------------------------------- |
| `.py`, `.pyi`                                                           | Python                                                                       |
| `.rs`, `.go`, `.c`, `.h`, `.cc`, `.cpp`, `.hpp`, `.java`, `.rb`, `.lua` | Respective source language; `.h` consistently C                              |
| `.js`, `.mjs`, `.cjs`, `.jsx`, `.ts`, `.tsx`                            | JavaScript, JSX, TypeScript, or TSX as declared by suffix                    |
| `.sh`, `.bash`, `.zsh`, recognized extensionless shell shebang          | Shell family, preserving explicit dialect where supported                    |
| `.json`, `.jsonl`, `.yaml`, `.yml`, `.toml`, `uv.lock`                  | JSON, YAML, or TOML                                                          |
| `.md`, `.markdown`, `.mdown`, `.mkd`, `.xprompt`, bare `README`         | Frontmatter-aware Markdown source                                            |
| `.rst`, including `README.rst`                                          | reStructuredText                                                             |
| `.html`, `.htm`, `.xml`, `.css`, `.tcss`, `.sql`                        | HTML, XML, CSS, or generic SQL                                               |
| `.diff`, `.patch`, trusted diff body                                    | Diff                                                                         |
| `Makefile`, `makefile`, `GNUmakefile`, `Dockerfile`                     | Make or Dockerfile                                                           |
| `.j2`, `.jinja`, `.jinja2`                                              | Jinja; this promises template delimiters, not full mixed-language parsing    |
| `.env`, `.env.*`                                                        | Shell assignment syntax                                                      |
| `.txt`, `.log`, `.sase`, `.gitignore`, `Justfile`, `justfile`           | Deliberately plain                                                           |
| Unknown or ambiguous input                                              | Existing plain/card behavior; explicit syntax remains available for raw text |

Suffix matching is case-insensitive; special basenames use explicit listed names.
Recognize extensionless Python and common shell shebangs, including `/usr/bin/env` and
`env -S`, without executing or locating the interpreter. Bound prefix inspection to 8
KiB and the shebang to its first line. Diff detection requires either a git diff header
or an adjacent old/new file header pair plus a valid hunk header; isolated `@@`, `---`,
prose, and JSON must not trigger it. Account for `/dev/null` in file header pairs. Never
perform an unbounded content guess.

### Source preservation and style precedence

Treat `PagerSection.plain_text` as the immutable coordinate system. Existing input
decoding/ANSI stripping is unchanged. All syntax offsets are Python character indices
into this string, not UTF-8 bytes or terminal display columns.

The syntax result is immutable role spans and a disposition such as highlighted, plain,
preserved, too-large, or failed. Build colored `Text` by styling a copy of the existing
source; never reconstruct the document from token values. Before accepting spans,
validate bounds, monotonic region order, and each token's source slice. Invalid/raising
top-level lexers discard the entire syntax result and leave the source visible; an
isolated Markdown child lexer can fail open for its fence.

Precedence is: original characters; existing producer styling or new syntax; existing
link target styles; jump capsules in derived render text; search match and current-match
styles in the search view. Syntax is skipped entirely for an ANSI source containing ESC,
a `Text` with a non-neutral base style or spans, or an opaque Rich renderable. The
explicit syntax override does not strip producer styles or force highlighting of
formatted cards. A plain unstyled `Text` is eligible only when an adapter explicitly
declares raw source provenance.

Link accents win over syntax foreground. Capsules retain their existing backgrounds and
boundaries. Dangling targets retain their existing dim treatment over syntax color.
Search still uses the existing logical unwrapped corpus without capsules; preserve
source styles and target accents without inserting any characters into that corpus, then
apply matches last. Never mutate cached `Text` while painting labels or matches.

### Appearance

Use a small semantic palette: keyword, type, function/decorator, constant/number,
string, comment, error, Markdown heading/emphasis/code, and diff added/deleted/hunk.
Allow separate roles where bold versus italic or diff header versus hunk behavior
requires it. Ordinary names, punctuation, and operators inherit body foreground. Do not
import xprompt tokenizers or fetch glossary/repository catalogs for this work.

Derive foreground colors from the active Textual theme using the house `HighlightStyle`
shape or a thin reuse of it. Use muted, readable comments with italic; bold headings;
correctly separated bold/italic Markdown emphasis; restrained inline-code color. Diff
additions/deletions use green/red foreground and retain their `+`/`-` text, with subdued
hunk headings. Error tokens get a modest underline or readable foreground, never a loud
filled background for incomplete source.

No syntax role paints an opaque background, gutter, border, or extra document row.
Reserve gold filled capsules for navigation. Test effective foreground contrast against
the actual body background at least 4.5:1 for the built-in dark/light themes used in
acceptance; avoid double-dimming comments below this threshold. Reuse hue families with
lower saturation when needed, rather than requiring disjoint colors from all link
accents. Palette adjustment is deterministic and cached by actual
foreground/background/theme values. Gracefully use readable neutral foreground if custom
colors cannot produce a suitable role.

Show a small muted language hint beside the current subject, for example `· py`, `· md`,
or `· diff`, only when syntax is actually enabled/prepared for that section. Use a
concise alias for other languages; there is no fixed three-cell promise. Drop the entire
hint before sacrificing the existing subject or position information at narrow widths.
Unknown, preserved, disabled, or skipped syntax adds no badge. Keep unknown/no-language
documents pixel-identical in the same host theme.

## Architecture and execution phases

### Language contract (`language_contract`)

Open `gh:sase-org/sase-core` with `/sase_repo` and use only the returned path. Read its
`AGENTS.md`; do not assume a sibling checkout or hardcode a numbered workspace. The core
inspection found no existing shared source-language resolver to extend.

Add a small pure Rust source-language module and additive versioned wire/API. A request
carries source category (raw file, stdin, Markdown document, diff, formatted), logical
filename if known, and a bounded prefix for permitted sniffing. Return an optional
canonical language, reason/provenance, and whether the name identifies a supported text
source. Formatted/non-text categories are ineligible. Keep all mapping, precedence,
shebang, and diff recognition rules here, with no file I/O or Pygments dependency.
Explicit engine aliases are validated in the frontend and can override the resulting
identity for eligible raw sections; this does not create a second automatic detection
policy.

Expose through `crates/sase_core_py`, following existing wire/binding registration and
schema tests. Add `src/sase/core/source_language_facade.py` plus wire rehydration as
needed, using `require_rust_binding` without a Python fallback or backend switch. Test
the real binding as well as pure Rust; stale/missing required bindings must produce the
project's normal clear dependency error, not a fake unknown language. Do not activate
this facade in current pager producers during this phase.

Define logical filename precedence for later adapters: original `source_path`, then
`vcs_relpath`, then the resolved file path. Use those fields only as filename hints;
read content from the already resolved artifact, never reopen the original source.
Preserve canonical refs, edit targets, auditing, and fragment line information.

Tests cover the mapping table, precedence, basename/suffix distinctions, extensionless
shebangs, positive and negative diff samples, bounded input, and JSON wire roundtrips.
Run the core repository's `just check`, including the PyO3 crate. Follow its release-plz
version ownership; do not manually bump Cargo versions. Verify Python facade integration
against the built binding, then run SASE `just check`.

### Syntax engine (`syntax_engine`)

Implement pager-local modules such as `syntax.py` and `syntax_theme.py`; use shared
style value types without importing unrelated semantic catalogs. Consume canonical
language names from the contract above. Engine-specific alias adaptation belongs here.
Add Pygments as a direct runtime dependency because the new production module imports it
directly; retain the existing renderer-stack pins and normal lockfile workflow rather
than relying solely on Rich's transitive dependency.

Use `get_tokens_unprocessed()` on lexers constructed with `stripnl=False`,
`ensurenl=False`, `stripall=False`, and `tabsize=0`. Explain the offset invariant in
code. Do not use `Syntax.highlight`, `Lexer.get_tokens`, Rich Markdown rendering,
subprocesses, or a TextArea replacement. Map token hierarchies through a closed, tested
role vocabulary; preserve Markdown fenced-language tokens.

Use a pager-owned offset-preserving Markdown composite, for example in
`pager/_markdown_syntax.py`, informed by `FrontmatterMarkdownLexer` but not its unsafe
nested-fence path or module-global cache. Partition source into leading YAML
frontmatter, prose, and fenced-code regions with original character offsets. Use
`MarkdownLexer(handlecodeblocks=False)` for prose and invoke each fenced language's raw
lexer directly, adding the region start to every validated local token offset. Reuse
Pygments YAML and Markdown rules rather than writing their grammars anew. The small
fence/region scanner is lexical presentation, not SDD parsing policy.

Cover backtick and tilde fences, up to three spaces of opening indentation,
same-character closing fences at least as long as the opener, a language alias with
optional following information, CRLF, and a closing fence at EOF. Unknown or unlabeled
fences keep a quiet code style; incomplete fences keep their source visible and avoid a
lexer crash. Recognize leading YAML fences for LF/CRLF and a closing fence at EOF
without normalizing any characters. Malformed/unclosed YAML frontmatter falls back to
ordinary source styling. Keep existing ACE consumers unchanged unless extracting a
compatible pure helper demonstrably reduces duplication; never mutate their global
lexer/token caches from a worker.

For arbitrary explicit or nested aliases, validate offset order as well as source
slices: token values must describe ordered non-overlapping regions of the supplied
source. Any composite lexer that violates the contract fails open for that region;
discard its partial spans. A Markdown child failure leaves that fence plain while
preserving valid surrounding Markdown. A top-level failure leaves the section plain.
Include a regression with a Python fence after a nonempty heading and a second fence
later in the document, asserting exact absolute positions of both inner keywords,
strings, and numbers.

Bound nested Markdown dispatch to three levels and share the enclosing section's
byte/line/span budget across children. A fence naming Markdown cannot recursively reset
budgets or bypass the fail-open contract.

Check limits before lexing: 80,000 UTF-8 bytes, 1,200 logical lines, and 4,096
characters per line. Define line counting consistently, with no extra line for a single
terminal newline. Also cap emitted spans at 20,000 per section. Stop and discard work
when a limit is exceeded; keep every source character, including oversized tails,
searchable. Empty/no-role outputs are valid. These are protection limits, not claimed
hard runtime bounds for arbitrary third-party regex lexers.

Produce immutable results and keep mutable lexer instances confined to one worker or
call. The token contract must work before any Textual app exists. Unit tests assert
exact source preservation for tabs, CRLF, absent/multiple trailing newlines, empty text,
Unicode (wide, combining, astral), malformed syntax, nested fences, frontmatter, and
deliberately invalid or failing lexers. Include producer-style suppression, span-budget,
and effective style/contrast tests.

This phase adds no active pager call site and changes no existing default theme.

### Reading surface (`reading_surface`)

Add optional, backward-compatible raw-source metadata to `PagerSection`, carrying
language/provenance and eligibility independently of `kind`, title, and subject ref. No
lexing or theme/config lookup belongs in `__post_init__`. Omitted metadata keeps today's
rendering. Existing production adapters still omit it until activation.

Extend composition to accept prepared source `Text` per section. Thread the same base
through `_section_renderable`, height measurement, and `render_section_with_labels`,
including its no-label branch. Preserve the current opaque-renderable path for
ineligible content. Do not store syntax in `body` or overwrite `_body_text`. Verify that
adding style changes neither wrapping nor row counts at either pager test width.

Add a small screen-owned preparation controller/mixin:

- Paint the existing body/chrome first. A thin synchronous `call_after_refresh` callback
  launches `spawn_pump_free_task`, which awaits `asyncio.to_thread` work. Never await
  lexing from Textual's message pump or do it during composition.
- Prepare the current section first, then remaining eligible sections sequentially. Keep
  one active preparation job per screen, with coalescing/last-request-wins state. Batch
  publication so many sections do not cause repeated full-body renders.
- Check byte/line/span limits in the worker. Confine worker-local lexers there and
  return immutable results. UI state/cache publication happens on the event loop.
- Cache completed outcomes, including skipped/error outcomes, by content digest and
  canonical lexer identity; cache styled bases additionally by theme signature and
  producer eligibility. Exclude width, scroll, label prefix, and search query. Do not
  hash all content per keypress: compute content identity once off-thread.
- Bound cross-document reuse to 24 sections and an aggregate span/source budget. Active
  prepared sections need stable ownership so ordinary scroll/resize does not re-lex
  evicted entries; bound a document to 200,000 prepared spans and keep additional
  sections plain when that budget is reached. Large unhighlighted source bodies are not
  duplicated into a syntax cache. Bound displayed `Text` variants to the current palette
  rather than retaining every previous theme.
- Tag requests with document generation, section identity/content identity, options, and
  theme signature. Reject stale completions after follow, refresh, history travel, or
  unmount. Cancellation of `to_thread` does not stop its thread: do not enqueue
  unbounded replacement work or assume cancellation proves work ended.
- Refresh only the applicable body/search view, preserving current scroll, search
  selection/query, label prefix, and trail state. If search is active when syntax
  finishes, repaint the search overlay rather than replacing it with normal body.
  Revalidate current state after every await. Cancel registered tasks at teardown.
- Separate palette/body invalidation from width invalidation. A theme change restyles
  cached tokens and requests any missing work without re-lexing. Refresh and
  back/forward reuse valid content; changed content never receives stale spans.

For search, add an optional styled-base provider to `VimSearchController` with an
unchanged plain default for all other hosts. Accept a defensive copy only when
`base.plain == self.corpus`; otherwise use the existing plain fallback. The pager
builds/caches this base from prepared section text, link target accents without
capsules, and exactly the existing `search_corpus` separators/newline conventions. Apply
match/current-match overlays last with the existing no-wrap/crop behavior. No lexing,
file access, corpus reconstruction, or tokenization per typed character. An explicit
repaint path handles late syntax completion and theme changes without restarting search
or resetting its selection. Cover the existing controller hosts and ACE zoom search with
compatibility tests.

Implement the active-theme palette input and optional subject hint. Keep theme selection
owned by the host app. Tests cover label/link style precedence (including dangling and
prefixes), attached-target behavior, follow/copy/editor line values, section navigation,
search enter/repeat/exit, delayed completion during search, theme changes, back/forward,
same identity with changed content, and unmount races. Use controlled worker barriers
and call counters rather than timing-based sleeps. Run applicable focused tests and SASE
`just check` before declaring this phase done.

### Activate and verify (`activate_and_verify`)

Wire metadata and resolved options into every real pager entry point:

- `path_section`/`document_from_paths` classify file content once during existing
  off-thread reads, recording language and original filename provenance.
- `resolve.py` propagates canonical document provenance and original artifact filenames
  rather than classifying content-addressed storage basenames. Use the shared
  classification to admit recognized source types missed by the old MIME gate. A bounded
  binary/NUL preview must prevent falsely named binaries from entering new text paths;
  preserve current media/directory/card routes. Keep any new shared eligibility rule in
  Rust, and keep I/O in the existing worker.
- `artifact_cli/read.py` classifies the body actually passed to its pager despite
  `_page_markdown`'s historical name. Use result kind/MIME/source name as genuine
  evidence; do not mark every artifact read as Markdown.
- `main/pager_handler.py`, `main/parser_pager.py`, and `cli_pager.py` handle syntax
  options and bounded unstyled-stdin diff detection. The generic no-document
  `page_or_print` case remains untyped except for reliable stdin evidence.
- ACE's `actions/hints/_files.py` supplies the normal config options without adding I/O
  to its UI callback. Commit manifests, bead detail, binary/media cards, and arbitrary
  pre-rendered output stay ineligible.
- Carry options through `SasePager` and embedded `PagerScreen`, including newly followed
  targets, while applying explicit language overrides only to the initial document's raw
  sections. Resolve config once off the render/key paths.

Add `-s/--syntax` with excellent help and the syntax/color/config precedence above.
Update `src/sase/default_config.yml` and the existing config validation/docs surface as
needed. Test unknown config values through established diagnostics. Make color policy
effective at the standalone app/console boundary; test both prepared source styles and
rendered terminal color output, since skipping syntax alone would leave the existing
`--color never` behavior misleading.

Add user examples to `docs/pager.md`, including automatic files, Markdown, piped diff,
stdin forced to YAML, and `--syntax none`. Explain conservative detection, preserved
ANSI, supported special filenames, config, and large-file plain fallback. Do not
advertise unrelated wrap, reload, or navigation fixes.

No temporary feature flag is needed if the first three phases remain unused capabilities
and activation is delivered only after their acceptance checks. If a worker instead
exposes an unfinished path in an earlier landed phase, the `sase_flags.md` beta-flag
procedure is mandatory; remove that scaffolding before final landing. The permanent
`pager.syntax` choice is not a feature flag.

## Final acceptance and verification

The feature is complete only when all these checks pass on the combined tree:

| Area              | Required evidence                                                                                                                                                                                                                                      |
| ----------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Source and output | Styled base equals canonical source exactly; target spans/copy values/editor lines unchanged; direct/plain output matches the pre-feature contract; no lexer invocation on direct paths                                                                |
| Detection         | Real CLI file and artifact resolution exercises every language family and special filename, mixed sections, original filenames behind opaque artifact paths, shebangs, negative diff samples, plain unknowns, and media/binary exclusions              |
| Precedence        | Effective Rich style assertions for code containing a file/ref/URL, capsule boundaries, ANSI and styled Text preservation, dangling links, Markdown emphasis, and active/current search matches                                                        |
| Controls          | CLI/config precedence, invalid aliases before stdin read, `auto`/`none`, color modes and `NO_COLOR`, embedded-host policy, and overrides versus followed/history documents                                                                             |
| Reliability       | Lexer errors/invalid offsets fail open; large/minified files retain their entire tails; stale worker results cannot repaint a different document, search state, or dismissed screen                                                                    |
| Caching           | Lexer counters unchanged across repeated scroll, resize, label prefix/window changes, and search typing; theme changes only restyle; content changes invalidate; skipped outcomes and aggregate budgets remain bounded                                 |
| Appearance        | Reviewed PNGs at 120x40 and 60x30 in the existing dark theme and a built-in light theme: Python with a link in a string, Markdown frontmatter/fenced Python, diff, active search, and unknown/prestyled controls; assert typographic styles separately |
| Responsiveness    | Controlled slow-worker test proves chrome/body and key handling work before completion; measured representative cold/warm open, scroll, and prefix latency with `tui_trace`; no added hot-path lexing/I/O and no attributable stall regression         |

Use the established `tests/pager/visual` fixture and PNG renderer, inspecting
actual/expected/diff artifacts before accepting changed goldens. Add only the fixtures
necessary to cover these distinct states; avoid a full combinatorial matrix.
Unknown/no-language controls must retain their prior geometry and colors in the same
theme. Search source separators and the no-capsule search convention must have
exact-text tests independently of PNGs.

For Python work, prepare the isolated environment with `just install` if necessary,
pointing `SASE_CORE_DIR` at the checkout returned by `/sase_repo` when building the new
binding. Use the matching `just rust-install` workflow; do not accidentally validate
against an old global wheel. Run focused pager/CLI/artifact/ACE-search tests and
`just check`. Run the relevant PNG lane through `just test-visual` with the supported
test-path arguments. Run the core repository's `just check`, which covers PyO3 as well
as pure Rust. Before landing the combined epic, run SASE `just check-full` only through
`/sase_monitor` with `TESTING`/`TESTED`; monitor any other verification that would block
a turn for a long time. Do not replace the required lane with targeted tests.

Release ordering is part of completion: make the additive core binding available before
activating its production callers, follow the repository's release-branch dependency
reconciliation to require a wheel exporting it, and verify the supported installed-wheel
path. Local editable success alone is insufficient. Do not invent a Python detection
fallback, swallow a missing binding as unknown syntax, or manually edit
release-plz-owned Cargo versions to work around ordering.

## Explicit limits

This work does not add a line-number gutter, a live syntax picker, new pager keys,
rendered/reflowed Markdown, external highlighter executables, semantic xprompt or
ProjectSpec lexers, or a general rewrite of file panels. Justfile remains plain until a
correct grammar is intentionally supported. Arbitrary unrecognized types can use an
explicit lexer; their automatic coverage can grow in the core table. Existing source
normalization, wrap controls, and reload semantics are preserved.

The caps and worker keep added work bounded and away from normal UI callbacks; they are
not a hard killable timeout for hostile regex grammars. No full-document virtualization
or above-cap progressive syntax engine is promised here. Full source text remains
visible/searchable under the existing pager's layout behavior.
