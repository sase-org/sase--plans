---
tier: epic
title: Derive glossary alias plurals automatically and hide them from ALIASES
goal: Glossary matching recognizes the plural form of every term and alias without
  it being configured, while the generated `ALIASES:` line lists only aliases the
  system cannot derive on its own and disappears entirely when nothing is left to
  list.
phases:
- id: core
  title: Derive plurals and display aliases in the Rust glossary domain
  depends_on: []
  size: medium
  description: 'core: add phrase pluralization to sase-core, split authored / effective
    / display alias lists, and expose the new display list on the glossary wire.'
- id: core-release
  title: Publish a sase-core-rs release containing the glossary change
  depends_on:
  - core
  size: small
  description: 'core-release: land the release-plz version bump for sase-core and
    confirm the new sase-core-rs wheel resolves from PyPI.'
- id: python
  title: Render display aliases and raise the core floor in sase
  depends_on:
  - core-release
  size: medium
  description: 'python: raise the sase-core-rs floor, carry the display alias list
    through the Python facade and LSP payload, render it in generated glossary memory,
    and regenerate the agent instruction files.'
proposed_by: bbugyi200.athena.wa.f0
create_time: 2026-08-09 08:17:19
status: wip
bead_id: sase-i3
---

- **PROMPT:** [prompts/202608/glossary_alias_plurals.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/glossary_alias_plurals.md)
- **BEAD:** [sase-i3](https://github.com/sase-org/sase--beads/blob/main/pages/sase-i3/README.md)

# Derive glossary alias plurals automatically and hide them from `ALIASES:`

## Problem

Every project glossary entry in `sase/sase.yml` currently has to spell out its own
plural as an alias just to get the plural highlighted:

```yaml
glossary:
  Agent Clan:
    aliases:
      - agent clans
    definition: >-
      An agent clan is a named, rootless container ...
```

Two things are wrong with that:

1. The plural is mechanically derivable from the term. Authors should not have to write
   it, and forgetting it silently loses highlighting.
2. Because the plural is configured, it is echoed back into the generated
   `ALIASES: agent clans` line in `sase/memory/glossary.md`, `AGENTS.md`, and every
   provider instruction shim. That line spends agent context restating something the
   reader already inferred from the term.

The requested behavior:

- Plural forms of the term and of every configured alias are added to the matched alias
  set automatically.
- Those plural forms are **not** shown in the rendered `ALIASES:` field.
- When an entry has no aliases left worth showing, the `ALIASES:` line **and the blank
  line above it** are not rendered at all.
- Plural forms keep being highlighted in the ACE prompt input widget and in external
  editors through the xprompt LSP.

## Current behavior

Glossary validation, normalization, and matching are Rust-owned in the sibling
`sase-core` repo; Python owns config discovery, presentation, and the memory generator.

- `crates/sase_core/src/glossary.rs` (sase-core)
  - `effective_aliases(entry)` returns the term first, then the configured aliases,
    normalized and deduplicated case-insensitively. It is used for three different
    purposes at once: validation iteration, the `effective_aliases` wire field, and the
    regex set built by `CompiledGlossaryCatalog::new()`.
  - `GlossaryEntryWire` carries `configured_aliases` (authored only) and
    `effective_aliases` (term + authored).
  - `GLOSSARY_WIRE_SCHEMA_VERSION` is `1`.
- `crates/sase_core_py/src/lib.rs` exposes `glossary_validate`, `glossary_catalog`, and
  `compile_glossary_catalog`.
- `crates/sase_xprompt_lsp/src/server.rs` does **not** recompute the catalog. It reads
  the launcher-generated `glossary_catalog.json`, deserializes `GlossaryEntryWire`
  values, and calls `CompiledGlossaryCatalog::new()` on them, so whatever
  `effective_aliases` Python wrote is exactly what the LSP matches on.
- sase (this repo)
  - `src/sase/core/glossary_facade.py` mirrors the wire records.
  - `src/sase/xprompt/glossary_catalog.py` builds entries from `sase/sase.yml`, compiles
    the matcher for ACE, and serializes the LSP payload in `_entry_to_wire()`.
  - `src/sase/integrations/xprompt_lsp.py` writes that payload to
    `~/.sase/xprompt_lsp/glossary_catalog.json` before launching the LSP.
  - `src/sase/main/init_memory/glossary.py` `_render_glossary_memory()` renders
    `{GLOSSARY_ALIASES_LABEL}: {aliases}` from `entry.configured_aliases`, already
    guarded by `if entry.configured_aliases:` so the label and its blank line are both
    skipped for an entry with no aliases.
  - `pyproject.toml` pins `sase-core-rs>=0.21.1,<0.22.0`.

So the matcher and the rendered label are both fed from the authored alias list, and
there is nowhere to put a derived alias that matches but does not print.

## Design

### Three alias lists instead of two

| List                | Contents                                                          | Consumers                                       |
| ------------------- | ----------------------------------------------------------------- | ----------------------------------------------- |
| authored (internal) | term first, then configured aliases (today's `effective_aliases`) | validation only                                 |
| `effective_aliases` | authored + derived plurals                                        | `CompiledGlossaryCatalog` regex set (ACE + LSP) |
| `display_aliases`   | configured aliases minus derivable plurals                        | generated glossary memory                       |

`configured_aliases` keeps its current meaning (normalized authored aliases) so no
existing consumer changes meaning underneath it.

Validation must iterate the **authored** list, not the effective one. Derived plurals
are not authored, so they must never produce a `blank_alias`, `duplicate_alias`, or
`alias_conflict` diagnostic pointing at a config line the user did not write.

### Pluralization rule

A deliberately conservative, dependency-free rule. Under-deriving costs one missing
highlight; mis-deriving pollutes the matcher and the LSP payload with a non-word.

Operate on the last whitespace-separated word of the already-normalized phrase, keeping
the preceding words verbatim:

1. If the phrase is empty, or the last word's final character is not an ASCII letter,
   derive nothing. (Guards `` `<suffix>` ``, `.md`, digits, punctuation.)
2. If the lowercased last word ends in `s`, derive nothing. This is what keeps an
   already-plural term such as `Agent Hoods` or `Agent Instruction Files` from producing
   `Hoodss` / `Filess`, and it deliberately gives up on `bus` / `class` rather than
   guessing.
3. Else if it ends in `x`, `z`, `ch`, or `sh`, append `es` (`Patch` → `Patches`,
   `Stitch` → `Stitches`).
4. Else if it ends in `y` preceded by a consonant, replace the `y` with `ies` (`Family`
   → `Families`, `Memory` → `Memories`).
5. Else append `s` (`Repo` → `Repos`, `xprompt` → `xprompts`).

Case of the original word is preserved; only the suffix is appended in lower case.
Matching is case-insensitive and derived plurals are never displayed, so the casing of a
derived form is not user-visible.

### Which derived plurals are accepted

Derived plurals are additive and must never displace anything authored:

- Skip a derived plural whose case-folded key is already claimed by any entry's authored
  alias (including the same entry's term or configured aliases, and including another
  entry's). Authored input always wins; the derivation stays silent instead of raising a
  new cross-entry `alias_conflict` the user cannot fix.
- Skip a derived plural whose key was already accepted for an earlier entry, so the
  result is deterministic in entry order.

`build_glossary_catalog()` runs `ensure_valid()` first, so the claim map can be built
from valid input without worrying about authored conflicts.

### Which aliases are displayed

An entry's `display_aliases` is its `configured_aliases` in authored order, minus any
alias `a` for which some **other** source `b` in `{term} ∪ configured_aliases` satisfies
`pluralize(b) == a` case-insensitively.

Comparing against every other source (rather than only against previously kept ones)
makes the result order-independent, and the rule cannot erase an entry entirely because
a plural is always strictly longer than its singular.

Worked against this repo's own `sase/sase.yml`:

| Term                    | Configured aliases                                      | Rendered `ALIASES:`                      |
| ----------------------- | ------------------------------------------------------- | ---------------------------------------- |
| Agent Clan              | agent clans                                             | _(line removed)_                         |
| Agent Family            | agent families                                          | _(line removed)_                         |
| Agent Hoods             | agent hood                                              | `agent hood`                             |
| Agent Lane              | agent lanes                                             | _(line removed)_                         |
| Agent Instruction Files | agent instruction file, agents.md files, agents.md file | `agent instruction file, agents.md file` |
| Agent Neighbors         | agent neighbor                                          | `agent neighbor`                         |
| Agent Tribe             | agent tribes                                            | _(line removed)_                         |
| Patch                   | patches                                                 | _(line removed)_                         |
| Project                 | projects                                                | _(line removed)_                         |
| Repo                    | repos, repository, repositories                         | `repository`                             |
| Stitch                  | stitches                                                | _(line removed)_                         |
| Workspace               | workspaces                                              | _(line removed)_                         |
| xprompt                 | xprompts                                                | _(line removed)_                         |
| xprompt Memory          | xprompt memories, memory xprompt, memory xprompts       | `memory xprompt`                         |
| xprompt Part            | xprompt parts                                           | _(line removed)_                         |
| xprompt Swarm           | xprompt swarms                                          | _(line removed)_                         |
| xprompt Workflow        | xprompt workflows                                       | _(line removed)_                         |

Twelve of seventeen entries lose their `ALIASES:` line and its blank line entirely; five
keep a shortened one. That table is the acceptance target for the regenerated files in
the `python` phase.

Note that for this repo's current config the derived plurals are all already configured,
so the accepted-plural set adds nothing new to the matcher. Highlighting behavior in ACE
and the LSP is therefore unchanged here — the derivation is what lets future entries
omit the plural, and what would preserve matching if the redundant aliases were later
deleted from the config.

### Why this belongs in Rust

Per the Rust core backend boundary rule, behavior a second frontend would have to match
is core logic. Both the ACE prompt widget and the xprompt LSP match through the same
compiled catalog, and the LSP matches on the `effective_aliases` Python serializes into
`glossary_catalog.json`. Deriving plurals anywhere in Python would either fail to reach
the LSP or leak the derived forms into `configured_aliases`, where the TUI preview and
the LSP hover would start printing them.

## Phase 1: Derive plurals and display aliases in the Rust glossary domain (`core`)

Use `/sase_repo` to open `sase-core` before reading or writing any file in it, and use
the path that skill prints. All work in this phase is in
`crates/sase_core/src/glossary.rs` unless noted.

1. Add a private `fn pluralize_phrase(phrase: &str) -> Option<String>` implementing the
   rule in **Pluralization rule** above. Place it next to `normalize_phrase`.
2. Rename the existing `fn effective_aliases(entry)` to `fn authored_aliases(entry)`
   with no behavior change, including the current quirk that a blank alias is pushed as
   an empty string so `validate_glossary_entries` can keep using
   `alias_index.saturating_sub(1)` to point back at the configured index. Update
   `validate_glossary_entries` to call `authored_aliases`; its diagnostics must be
   byte-identical to today's for every input.
3. Add `#[serde(default)] pub display_aliases: Vec<String>` to `GlossaryEntryWire`,
   directly after `configured_aliases`. `#[serde(default)]` is required: the xprompt LSP
   deserializes this struct from an on-disk payload that a not-yet-upgraded sase may
   have written.
4. Do **not** change `GLOSSARY_WIRE_SCHEMA_VERSION`. `load_glossary_catalog()` in
   `crates/sase_xprompt_lsp/src/server.rs` hard-gates on `schema_version == 1` and
   degrades to no glossary semantics on a mismatch, so a bump would silently disable
   editor glossary support for any mixed-version pair. Adding a defaulted field is
   backward and forward compatible in both directions: serde ignores unknown fields, so
   an older LSP binary still reads a newer payload.
5. Rework `catalog_from_entries` to compute all three lists:
   - `configured_aliases`: unchanged.
   - `display_aliases`: derived from the normalized `configured_aliases` per **Which
     aliases are displayed**.
   - `effective_aliases`: `authored_aliases(entry)` with empty strings filtered out,
     followed by accepted derived plurals per **Which derived plurals are accepted**.
     Build the cross-entry claim map from every entry's authored alias keys in a first
     pass, then walk entries in order accepting plurals in a second pass.
6. Update the existing `builds_effective_aliases_with_term_first` unit test and add unit
   tests covering, at minimum:
   - `pluralize_phrase` for the `s` / `x` / `z` / `ch` / `sh` / consonant-`y` /
     vowel-`y` / default branches, the already-plural skip, and the non-letter-final
     skip;
   - multiword phrases pluralizing only the last word (`agents.md file` →
     `agents.md files`);
   - a derived plural appearing in `effective_aliases` when it is not configured;
   - a derived plural being skipped when another entry already claims that key, with no
     new diagnostic emitted;
   - `display_aliases` dropping a term-derived plural, dropping an alias-derived plural
     regardless of authored order, and keeping a non-derivable alias;
   - `display_aliases` being empty when the entry's only alias is the term's plural;
   - `validate_glossary_entries` producing the same diagnostics as before for the blank,
     duplicate, multiline, and conflicting alias cases.
7. Update the doc comments in `glossary.rs` to state that `effective_aliases` is the
   match set including derived plurals while `display_aliases` is the render set.
8. Leave `glossary_hover_markdown` in `crates/sase_xprompt_lsp/src/server.rs` reading
   `configured_aliases`. Editor hover showing exactly what the config says is correct;
   changing it is not part of this request.
9. Do not hand-edit any crate version or `Cargo.toml` version field — release-plz owns
   those.

Verification, from a sase workspace (these recipes drive the sase-core checkout):

```bash
just rust-fmt
just rust-check   # rust-fmt-check + rust-clippy + rust-test
```

and in the sase-core checkout:

```bash
cargo test --workspace glossary
```

Acceptance: `cargo test --workspace` is green; validation diagnostics are unchanged for
all pre-existing inputs; a term with no configured aliases now matches its own plural.

## Phase 2: Publish a sase-core-rs release containing the glossary change (`core-release`)

The Python phase raises `pyproject.toml`'s `sase-core-rs` floor, which requires the
version to exist on PyPI first.

1. Confirm the `core` phase commit is on `sase-core` `master`.
2. Find the release-plz release pull request for `sase-core` and confirm its changelog
   entry covers the glossary change and that the version bump is inside the current
   `<0.22.0` ceiling declared by sase's `pyproject.toml`. If release-plz proposes a bump
   that leaves that range, stop and raise it with the project owner rather than editing
   either side unilaterally — the ceiling change is a separate decision.
3. Merge that release PR if the project owner has not already, then wait for the publish
   workflow to finish.
4. Record the exact published version and confirm it resolves:

   ```bash
   uv pip download --no-deps --dest /tmp/core-release-check "sase-core-rs==<version>"
   ```

Acceptance: a `sase-core-rs` version containing the `core` phase change is installable
from PyPI, and its number is recorded on this phase's bead for the `python` phase to
consume.

## Phase 3: Render display aliases and raise the core floor in sase (`python`)

All work in this phase is in this repo.

1. Raise the floor in `pyproject.toml` to the version published in `core-release`, e.g.
   `"sase-core-rs>=<version>,<0.22.0"`. Keep the existing ceiling.
2. `src/sase/core/glossary_facade.py`: add `display_aliases: tuple[str, ...]` to
   `GlossaryEntry` immediately after `configured_aliases`, and read it in `from_wire`
   with a strict `payload["display_aliases"]`. Do not add a tolerant fallback to
   `configured_aliases` — the floor bump is what guarantees the field, and this module's
   sibling `src/sase/core/rust.py` deliberately fails fast on a stale wheel rather than
   silently degrading.
3. `src/sase/xprompt/glossary_catalog.py`: emit `"display_aliases"` in `_entry_to_wire`
   so the LSP payload round-trips the full record.
4. `src/sase/main/init_memory/glossary.py`: in `_render_glossary_memory()`, read
   `entry.display_aliases` instead of `entry.configured_aliases` for both the guard and
   the joined value. The existing `if` already suppresses the label and the blank line
   above it together, so no other change is needed for the empty case — keep that
   structure rather than rewriting it.
5. Leave `_glossary_preview_markdown` in `src/sase/ace/tui/widgets/_prompt_glossary.py`
   reading `configured_aliases`. The ACE `K` preview is a config-inspection surface, and
   leaving it alone also keeps `tests/ace/tui/widgets/test_prompt_glossary.py` and the
   ACE PNG snapshots stable.
6. Update the tests that construct `GlossaryEntry` positionally or by keyword, all of
   which now need the new field:
   - `tests/test_core_glossary_facade.py` (wire payload fixture and assertions)
   - `tests/xprompt/test_glossary_catalog.py`
   - `tests/ace/tui/widgets/test_prompt_glossary.py`
   - `tests/ace/tui/visual/_ace_prompt_png_snapshot_helpers.py`
7. Extend `tests/main/test_init_memory_glossary.py`:
   - the `Agent Clan` case with `aliases: [agent clans, clan]` now renders
     `ALIASES: clan`; assert that, and assert `agent clans` is absent from the rendered
     note;
   - add a case whose only alias is the term's plural and assert the generated note
     contains no `ALIASES:` line for that entry and no doubled blank line before its
     definition;
   - keep the existing negative assertion that title-case `Aliases:` never appears.
8. Add a Python-level regression test proving the derived plural still matches through
   the compiled catalog — build a catalog for a term with **no** configured aliases,
   scan text containing its plural, and assert a span is returned. Put it next to the
   existing facade tests in `tests/test_core_glossary_facade.py`. This is the test that
   would catch a future core downgrade breaking the highlighting half of the request.
9. Regenerate the derived memory artifacts. These are SASE memory and agent instruction
   files: the user explicitly requested this rendering change, which authorizes the
   mandatory regeneration, but every one of these files must be produced by the
   generator and never hand-edited.

   ```bash
   just install          # rebuilds sase_core_rs from the sase-core checkout
   sase memory init
   sase memory init --check
   ```

   Expected to change: `sase/memory/glossary.md`, `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`,
   `OPENCODE.md`, `QWEN.md`. Diff them against the **Which aliases are displayed** table
   — twelve entries lose their `ALIASES:` line plus the blank line above it, and five
   keep a shortened one. Any other drift means the derivation rule was implemented
   differently than specified.

10. Documentation:
    - `docs/configuration.md` `### glossary`: update the `aliases` row and the
      `sase memory init` paragraph to say that the plural of the term and of each alias
      is matched automatically, that derivable plurals are omitted from the rendered
      `ALIASES:` line, and that the line is omitted entirely when nothing remains.
      Update the example block so it no longer teaches authors to write
      `aliases: [agent clans]`.
    - `src/sase/config/sase.schema.json`: extend the `glossaryEntry.aliases` description
      to note that plurals are derived automatically and need not be listed.
    - `docs/xprompt.md` (the LSP glossary paragraph around the semantic-token
      description): one sentence that derived plurals are matched like configured
      aliases.

Verification:

```bash
just install
just check-full
just test-visual
pytest tests/main/test_init_memory_glossary.py tests/test_core_glossary_facade.py \
       tests/xprompt/test_glossary_catalog.py tests/ace/tui/widgets/test_prompt_glossary.py
sase memory init --check
grep -rn "^Aliases:" AGENTS.md CLAUDE.md GEMINI.md OPENCODE.md QWEN.md sase/memory/glossary.md
```

`just check-full` is required rather than `just check` because this touches checked-in
`AGENTS.md` and memory files, and `just test-visual` because the visual snapshot helper
constructs `GlossaryEntry` directly. The final `grep` must return no matches.

Acceptance: the regenerated files match the table, `sase memory init --check` exits 0,
and the compiled-catalog test proves an underived plural is still matched.

## Explicitly out of scope

- `sase/sase.yml`'s glossary config. The redundant plural aliases in it become invisible
  in the generated output and are re-derived for matching, so deleting them is a pure
  no-op on every rendered artifact. Deleting them is reasonable hygiene later; doing it
  in this epic only adds review surface. (One cosmetic difference if it is ever done:
  `GlossarySpanWire.alias` would report the derived casing, e.g. `Agent Clans` rather
  than `agent clans`.)
- Other SASE projects' glossaries (chezmoi and the plugin repos). They pick up the new
  rendering the next time their own `sase memory init` runs; do not regenerate them
  here.
- `_glossary_preview_markdown` in ACE and `glossary_hover_markdown` in the xprompt LSP.
  Both keep printing `configured_aliases`. Switching them to `display_aliases` is a
  defensible follow-up but changes editor UX that was not part of this request.
- Singularization. Only plurals are derived, so an entry whose term is already plural
  (`Agent Hoods`, `Agent Instruction Files`, `Agent Neighbors`) still has to configure
  its singular alias by hand.
- Irregular plurals (`index` → `indices`, `matrix` → `matrices`) and the `s`-final words
  the rule deliberately skips (`bus`, `class`). Configure those as explicit aliases.

## Risks

- **Silently wrong derivation.** A bad rule injects a non-word into the matcher and into
  the LSP payload, where it is invisible in the docs and hard to trace. Mitigated by the
  conservative rule, by deriving nothing rather than guessing, and by the per-branch
  unit tests in phase 1.
- **New validation errors on previously valid configs.** If validation were left
  iterating the derived set, a derived plural colliding across two entries would fail
  someone's config at a line they never wrote. Phase 1 step 2 (validate over authored
  aliases) plus the skip-on-claim rule are what prevent this; the phase 1 test list pins
  both.
- **Matcher cost in the prompt render path.** `CompiledGlossaryCatalog::candidate_spans`
  runs every alias regex over the text, so accepted plurals grow the pattern count by at
  most one per authored alias. `_build_highlight_map` already refuses to scan past
  `_MAX_OVERLAY_BYTES` / `_MAX_OVERLAY_LINES`, and for this repo's config the accepted
  set does not grow at all because the plurals are already configured. No perf work is
  planned; if `SASE_TUI_PERF=1` key-to-paint p95 regresses past 16 ms on the prompt tab
  after the change, treat that as a finding to report rather than something to fix
  inside this epic.
- **Cross-repo ordering.** The `python` phase cannot land its floor bump before the
  `core-release` phase publishes. It can still be developed and tested locally at any
  point after `core`, because `just install` builds the extension from the sase-core
  checkout rather than from PyPI.

## Definition of done

- A glossary term with no configured aliases is highlighted in its plural form in the
  ACE prompt input and by the xprompt LSP.
- Derived plurals never appear in `ALIASES:` in `sase/memory/glossary.md`, `AGENTS.md`,
  or any provider instruction shim.
- Configured aliases that the system can derive are omitted from `ALIASES:`, and an
  entry with nothing left to show renders neither the `ALIASES:` line nor the blank line
  above it.
- `validate_glossary_entries` emits the same diagnostics it does today for every input
  that is valid or invalid today.
- `sase memory init --check` exits 0 and the six regenerated files match the table in
  **Which aliases are displayed**.
- `just rust-check` and `cargo test --workspace` pass in sase-core; `just check-full`
  and `just test-visual` pass in sase.
