---
tier: tale
title: Core and reference memory vocabulary
goal:
  SASE memory tiers are called core memory and reference memory everywhere — Rust wire,
  note frontmatter, AGENTS.md anchors, templates, skills, CLI help, docs, and prose —
  the legacy `short`/`long` spellings and `Tier 1 (short-term)`/`Tier 2 (long-term)`
  anchors are accepted forever on read, and every note in this repo plus the home root
  has been migrated in place by `sase memory init`.
size: medium
proposed_by: bbugyi200.athena.sase-sq.1
bead: sase-sq.1
---

- **PARENT:** [202608/memory_webs.md](memory_webs.md)
- **BEAD:**
  [sase-sq.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-sq/sase-sq.1.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-sq.1](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-sq.1.md)
- **COMMITS:**
  - [f6eedd9](https://github.com/sase-org/sase-core/commit/f6eedd98fbb6e72cb0adeb7fd40a71ff5b47906e)
    — feat(memory): support core and reference memory tiers

# Plan: Core and reference memory vocabulary

Implements phase `tiers` (bead `sase-sq.1`) of epic `sase-sq` · Memory webs and strands
(`plan:202608/memory_webs.md`). Read that epic plan's **Vocabulary**, **Rust boundary**,
and **tiers** sections before starting; this plan is the executable expansion of them.

## Goal

Rename "short-term memory" to **core memory** and "long-term memory" to **reference
memory** across the whole product surface, and rename the frontmatter values `short` →
`core` and `long` → `reference`. Old spellings keep parsing **forever**; old `AGENTS.md`
anchors keep parsing **forever**. Nothing about how memory works changes — only what it
is called and what `type:` says on disk.

This is a wide, mechanical rename with three non-mechanical parts, and those three are
where the whole risk lives:

1. A coordinated `sase-core` wire change that must land and release **before** this
   repo's change reaches anyone.
2. Compatibility that is **additive only** — every reader must accept both spellings,
   because generated files in unmigrated trees still carry the old ones.
3. A data migration that rides `sase memory init`'s existing frontmatter-update path and
   therefore also touches the chezmoi home root.

## Context: what this phase does not do

- **No identifier sweep.** The bead names "the Rust tier wire, note frontmatter,
  AGENTS.md anchors, templates, skills, docs, and prose". Python identifiers such as
  `_short_memory_bodies`, `inlined_short_memory_files`, `GeneratedLongMemoryNote`, and
  `AmdLongMemoryDescriptionUpdate` keep their names, except the two anchor regex
  constants and the two README template variables that the epic plan names explicitly.
  Renaming ~40 more symbols would multiply the review surface of an already-wide diff
  for no behavior change. Record it as a `PROPOSED FOLLOW-UP:` instead.
- **No `bob-cli` regeneration.** It needs the _released_ `sase` on `PATH`, which does
  not exist until this lands. Legacy spellings are accepted forever, so leaving
  `bob-cli` on `type: short|long` is correct, not broken. Follow-up note.
- **No `sase-core-rs` floor bump.** `tools/ratchet_core_window` resolves against PyPI,
  and the release that carries this change is not published while the phase is being
  implemented. Follow-up note; `just install` builds `sase_core_rs` from the local
  `sase-core` checkout, so every gate here passes without it.
- **No image regeneration.** `src/sase/memory/assets/memory-directory-map.prompt.md`
  describes the labels baked into `memory-directory-map.png`. Editing the prompt without
  regenerating the PNG would desync them. Leave both; follow-up note.
- **No blog-post edits.** `docs/blog/posts/*.md` are dated, published narrative. They
  keep saying what they said.

## Step 1 — `sase-core` (do this first, verify it first)

Open the linked checkout with the `/sase_repo` skill and work there:

```bash
sase repo open sase-core -r "sase-sq.1: rename memory tiers to core/reference"
```

`release-plz` owns versions in that repo: **do not** edit `[workspace.package].version`,
crate versions, or path-dependency pins. Use a Conventional Commit subject.

### 1.1 `crates/sase_core/src/content_layout.rs`

Rename the variants and widen `parse`. The serde aliases are the load-bearing part:
`MemoryTierWire` is deserialized from `MobileXpromptCatalogEntryWire.memory_type`
(`crates/sase_core/src/host_bridge.rs`), which Python produces, so a payload written by
an older `sase` must still deserialize.

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum MemoryTierWire {
    #[serde(alias = "short")]
    Core,
    #[serde(alias = "long")]
    Reference,
}

impl MemoryTierWire {
    pub fn parse(value: &str) -> Option<Self> {
        match value.trim() {
            "core" | "short" => Some(Self::Core),
            "reference" | "long" => Some(Self::Reference),
            _ => None,
        }
    }

    pub fn as_str(self) -> &'static str {
        match self {
            Self::Core => "core",
            Self::Reference => "reference",
        }
    }
}
```

Update the enum doc comment to say core/reference. In `memory_note_issue`, the rejection
message becomes "… must declare `type: core` or `type: reference` to be an xprompt
memory".

### 1.2 Call sites (mechanical variant rename)

`crates/sase_core/src/xprompt_catalog.rs`, `crates/sase_core/src/editor/hover.rs`,
`crates/sase_core/src/editor/completion.rs`, `crates/sase_core/src/editor/definition.rs`
— all `MemoryTierWire::Short` → `::Core`, `::Long` → `::Reference`. `hover.rs` renders
`tier \`{as_str()}\``, so its expected-hover assertions now read `tier
\`reference\``. Also update the `memory_type`doc comment on`MobileXpromptCatalogEntryWire`in`host_bridge.rs`.

### 1.3 Gateway contract

`crates/sase_gateway/src/contract.rs` describes `memory_type` as "`short` or `long`".
Change it to name the current values and the accepted legacy ones:

> `string|null; \`core\` or \`reference\` (legacy \`short\`/\`long\` still accepted) for
> an xprompt memory referenced as \`#memory/<stem>\`, absent otherwise; a non-null value
> means \`kind\` is \`memory\``

Then regenerate the snapshot so the contract test passes:

```bash
cargo run -p sase_gateway -- --contract-out crates/sase_gateway/contracts/api_v1/mobile_api_v1.json
```

### 1.4 Rust tests

In `content_layout.rs`'s test module, extend the existing tier test and add a serde
round-trip test:

- `parse("core")`, `parse("reference")`, `parse("short")`, `parse(" long ")` all
  resolve; `parse("dynamic")` is `None`.
- `MemoryTierWire::Reference.as_str() == "reference"`,
  `MemoryTierWire::Core.as_str() == "core"`.
- `serde_json::to_string(&MemoryTierWire::Core)` is `"core"`.
- `serde_json::from_str::<MemoryTierWire>("\"short\"")` is `Core` and `"\"long\""` is
  `Reference` — the legacy-payload guarantee.
- `memory_note_issue("m.md", "m", Some("core"))` is `None`; `Some("short")` is still
  `None`; `Some("dynamic")` yields `InvalidNoteType` and the message names `type: core`
  / `type: reference`.

### 1.5 Verify

```bash
cd <sase-core checkout> && ./scripts/check.sh
```

Never `cargo test -p sase_core` alone — that skips the `sase_core_py` binding tests.

## Step 2 — `sase` install picks up the local core

```bash
just install
```

The Justfile builds `sase_core_rs` from the linked `sase-core` checkout and lifts the
`pyproject.toml` version window for dev installs, so the new binding is live locally
with no PyPI release. If `just install` reports the checkout is behind the declared
floor, stop and re-read — that means step 1 was not built, not that the floor needs
editing.

## Step 3 — Python data model normalization

`src/sase/memory/notes.py` is the choke point. Normalize on parse so **no downstream
comparison site ever has to know a legacy value exists**.

```python
MemoryNoteType = Literal["core", "reference"]

_LEGACY_NOTE_TYPES = {"short": "core", "long": "reference"}
_VALID_NOTE_TYPES = frozenset({"core", "reference"})
```

In `parse_memory_note_text`, after `parsed_type = _normalized_scalar(raw_type)`, map a
legacy spelling to its canonical one before deciding `type_source`:

- `type: short` → `note.type == "core"`, `type_source == "frontmatter"`.
- `type: core` → `note.type == "core"`, `type_source == "frontmatter"`.
- `type: bogus` → `note.type == "bogus"` (unchanged raw value),
  `type_source == "invalid"`. Keep returning the raw value for invalid types so
  diagnostics still show what the file actually said.

Export a small public helper,
`normalize_memory_note_type(value: str | None) -> str | None`, from `notes.py` and add
it to `__all__`; `xprompt/loader_memory.py` and `memory/mutation_validate.py` both need
the same mapping and must not each hand-roll it.

Then swap the value literals at every comparison site. Because parsing normalizes, each
is a one-word change, not a compatibility branch:

| File                                                                                               | What changes                                                                                                                                                                                                                  |
| -------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `src/sase/memory/notes.py`                                                                         | `_children_of`, `_render_memory_note_references`, `render_long_memory_sections` compare `"reference"`; docstrings say "reference notes"                                                                                       |
| `src/sase/memory/read_log.py`                                                                      | `== "core"` guard; `!= "reference"` guard; error text "memory file is not a reference memory note"                                                                                                                            |
| `src/sase/memory/render.py`                                                                        | `note.type == "reference"`; `note.type or "reference"`; docstring                                                                                                                                                             |
| `src/sase/memory/mutation.py`                                                                      | `resolved_type = "core" if note.type == "core" else "reference"`                                                                                                                                                              |
| `src/sase/memory/mutation_validate.py`                                                             | accept `core`, `reference`, `short`, `long` on input via `normalize_memory_note_type`; error text "memory note type must be core or reference"; every downstream `parsed_type ==` comparison; `note_type="reference"` default |
| `src/sase/memory/proposals/review.py`                                                              | `!= "reference"`, `note_type="reference"`                                                                                                                                                                                     |
| `src/sase/memory/inventory_reachability.py`                                                        | `_is_short_memory_note` compares `"core"`; the three `"long"` comparisons compare `"reference"`; docstrings                                                                                                                   |
| `src/sase/memory/cli_show.py`, `cli_write.py`, `cli_review.py`, `cli_list.py`, `review_tui/app.py` | docstrings and printed prose                                                                                                                                                                                                  |
| `src/sase/amd/_memory.py`                                                                          | `note.type == "reference"` (×3), `note.type == "core"`, `note_type="reference"`, `type="reference"`                                                                                                                           |
| `src/sase/main/init_memory/root_rendering.py`                                                      | sort key `{"core": 0, "reference": 1}`                                                                                                                                                                                        |
| `src/sase/main/init_memory/root_rendering_notes.py`                                                | `note_type="core"` (×2), `note_type="reference"`                                                                                                                                                                              |
| `src/sase/main/init_memory/root_rendering_task_types.py`                                           | `note_type="core"`, `!= "core"`                                                                                                                                                                                               |
| `src/sase/main/init_memory/root_rendering_artifact_relations.py`                                   | `note_type="core"`                                                                                                                                                                                                            |
| `src/sase/xprompt/models.py`                                                                       | `MemoryType = Literal["core", "reference"]`                                                                                                                                                                                   |
| `src/sase/xprompt/loader_memory.py`                                                                | `_memory_type` routes through `normalize_memory_note_type`; the defense-in-depth message says `type: core` or `type: reference`                                                                                               |
| `src/sase/ace/tui/memory_panel_catalog.py`                                                         | four type comparisons                                                                                                                                                                                                         |
| `src/sase/ace/tui/modals/memory_panel_add.py`                                                      | `{"core", "reference"}`, `initial_type: str = "reference"`, select options `("core — Tier 1, always loaded", "core")` and `("reference — Tier 2", "reference")`, both `== "core"` guards                                      |
| `src/sase/ace/tui/modals/memory_panel_delete.py`                                                   | `tier = "1 (core)" if note.type == "core" else "2 (reference)"`; the `== "core"` guard                                                                                                                                        |
| `src/sase/ace/tui/modals/memory_panel_rendering.py`                                                | `!= "reference"`, `== "core"` (×2)                                                                                                                                                                                            |
| `src/sase/ace/tui/modals/memory_panel_actions.py`                                                  | `note.type in {"core", "reference"} else "reference"`                                                                                                                                                                         |

The `TIER 1 · always loaded` / `TIER 2` badge strings and the `_TIER1_MARK` /
`_TIER2_MARK` glyphs stay exactly as they are — Tier 1 and Tier 2 remain the anchor
names.

## Step 4 — Document anchors: add, never replace

`src/sase/amd/_agents_doc.py` parses **already generated** documents. A newer `sase`
that stopped accepting `Tier 1 (short-term) Memory` would fail to parse every
not-yet-reinitialized project's `AGENTS.md` and every provider shim. Widen, then rename
the constants:

```python
_CORE_SECTION_RE = re.compile(
    r"^##\s+(?:\d+(?:\.\d+)*\.?\s+)?Tier 1 \((?:short-term|core)\) Memory$"
)
_REFERENCE_SECTION_RE = re.compile(
    r"^##\s+(?:\d+(?:\.\d+)*\.?\s+)?Tier 2 \((?:long-term|reference)\) Memory$"
)
```

`_SHORT_MEMORY_BULLET_RE`, `_SHORT_MEMORY_HEADER_RE`, `_LONG_MEMORY_ENTRY_RE`, and
`_LONG_MEMORY_SECTION_RE` are **unchanged** — they match `memory/<file>.md` paths, not
tier words. Update the two usages of the renamed constants and the module's comment
prose.

In `src/sase/amd/_memory.py::_render_managed_agents`, the two structural-anchor
assertions must now name the emitted anchors:

- "rendered AGENTS template is missing structural anchor `## Tier 1 (core) Memory`"
- "rendered AGENTS template is missing structural anchor `## Tier 2 (reference) Memory`"

## Step 5 — Templates and generated prose

### 5.1 `src/sase/amd/templates/AGENTS.template.md`

```markdown
## Tier 1 (core) Memory

## Tier 2 (reference) Memory
```

Everything else in the template is untouched.

### 5.2 `src/sase/mdtemplates.py` — allow a variable without requiring it

`render_markdown_template` currently treats `required_variables` as both the required
set and the allowed set: `unknown = variables - required_variables` is an error. A user
whose overridden `memory-README.template.md` says `{{ short_notes }}` would therefore
fail twice — missing `core_notes`, unknown `short_notes`.

Add an `optional_variables: Set[str] = frozenset()` keyword-only parameter, default
empty so every other caller is unaffected, and compute:

```python
missing = sorted(required_variables - variables)
unknown = sorted(variables - required_variables - optional_variables)
```

### 5.3 `src/sase/main/init_memory/root_rendering.py`

Split the template variable set:

```python
_MEMORY_README_TEMPLATE_VARS = frozenset(
    {"memory_notes", "total_notes", "total_lines", "total_tokens"}
)
_MEMORY_README_OPTIONAL_TEMPLATE_VARS = frozenset(
    {"core_notes", "reference_notes", "short_notes", "long_notes"}
)
```

Pass **both** spellings in the render context (`core_notes` and `short_notes` hold the
same count; `reference_notes` and `long_notes` likewise) and pass the optional set
through. The packaged template uses only the new names. A comment must say why the old
names are still fed: user template overrides.

### 5.4 `src/sase/main/init_memory/templates/memory-README.template.md`

Rewrite the vocabulary and, per the epic's two-axis rule, state both axes explicitly so
later phases have something to attach "web" and "strand" to:

- Intro: "compact, always-loaded **core** notes from detailed **reference** notes".
- "`type: core` notes are Tier 1 context…", "`type: reference` notes are detailed
  reference material for Tier 2…".
- Frontmatter schema: "`type`: `core` for always-loaded notes or `reference` for
  read-on-demand notes. The legacy values `short` and `long` are still accepted and mean
  the same thing." Keep `parent:` describing a reference note nested under another
  reference note.
- Add a short paragraph under the frontmatter schema naming the two axes: what a memory
  **is** (today: a note) and how it **renders** (`core` or `reference`) are independent
  properties; `type:` declares rendering only.
- Statistics: `- Core notes: {{ core_notes }}` and
  `- Reference notes: {{ reference_notes }}`.
- The `sase memory read`/`write` command bullets say "reference note".

### 5.5 `src/sase/amd/inline_memory.py`

Module docstring and the `## Tier 1 (short-term)` mention become core; the validator
docstring says "core memory section".

## Step 6 — Prose sweep

125 occurrences of "short-term"/"long-term" across `src/`, `tests/`, `sase/memory/`, and
`docs/`. Excluded: `docs/blog/posts/*.md` and
`src/sase/memory/assets/memory-directory-map.prompt.md` (see the non-goals above).

Sweep, at minimum:

- `src/sase/main/parser_memory.py` (10 occurrences — `read`, `show`, `write`, `review`,
  and `log` help and description strings)
- `src/sase/main/parser.py` (the `memory` group description)
- `src/sase/notifications/senders.py`, `src/sase/memory/review_tui/app.py`,
  `src/sase/memory/cli_write.py`, `cli_show.py`, `cli_review.py`, `render.py`,
  `read_log.py`, `notes.py`, `src/sase/amd/_shared.py` docstrings
- `docs/memory.md` (12), `docs/init.md` (11), `docs/configuration.md` (4), `docs/cli.md`
  (3), `docs/xprompt.md` (2), `docs/architecture.md`, `docs/index.md`,
  `docs/notifications.md`
- `sase/memory/README.md` is generated — do not hand-edit it; step 9 regenerates it.

In docs, also update the two literal anchor strings (`## 1. Tier 1 (short-term) Memory`
in `docs/memory.md`, `## Tier 2 (long-term) Memory` in `docs/init.md`) and the
`type: short` mention in `docs/init.md`, and say once, where `type:` is introduced, that
`short` and `long` remain accepted.

Grep gate before moving on — outside the two excluded paths this must come back empty:

```bash
grep -rn "short-term\|long-term" --include='*.py' --include='*.md' src/ tests/ docs/ sase/memory/ \
  | grep -v 'docs/blog/posts/' | grep -v 'memory-directory-map.prompt.md'
```

`AGENTS.md`, `CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`, `QWEN.md`, and `CHANGELOG.md` also
contain the old anchors. The first five are generated by step 9; `CHANGELOG.md` is
history and stays.

## Step 7 — The `/sase_memory_read` skill

Rewrite `src/sase/xprompts/skills/sase_memory_read.md`. It is a generated skill source;
deployment to managed locations is a separate, post-land `sase skill init` step (see
`sase/memory/generated_skills.md`) and is **not** part of this phase.

Content requirements:

- `name:` stays `sase_memory_read`. The `description:` frontmatter says "Guide audited
  SASE reference memory reads through `sase memory read`. Use when instructions require
  reading SASE reference memory or mention the reference-memory read procedure." Keep
  the old phrase "long-term memory" out of it — but keep the description recognizably
  about the same command, since it is the routing text agents match against.
- State the two axes explicitly, in the epic's words: what a memory **is** and how it
  **renders** are independent; today every memory is a note, and `type:` declares only
  whether it renders as core memory (always loaded) or reference memory (read on
  demand).
- Say that core memory cannot be read with this command because it is already in
  context.
- Preview the coming grammar in one short "Coming soon" line —
  `sase memory read <web>:<keyword>` for keyed collections — flagged as not yet
  available, so the skill does not teach a command that errors today.
- Keep the existing rules, `## Command` block, and both examples, retermed.

## Step 8 — Data migration through `sase memory init`

`src/sase/amd/_memory.py::_long_memory_description_updates` already rewrites reference
notes' frontmatter with `apply_memory_frontmatter` and feeds the result into
`root_rendering.py` as a `MemoryExpectedFile(stale_operation="update")`, so it already
shows up in `--check` and `--diff`. Generalize it rather than building a second path.

Rename it `_memory_frontmatter_updates` (this one **is** in scope: its current name
would be a lie) and change the loop:

- Drop the `if note.type != "long": continue` guard; iterate core and reference notes.
- Keep skipping paths in `generated_long_notes` — those are rendered wholesale by the
  expected-file path and must not be double-planned.
- Skip notes whose `type_source` is `"invalid"` or `"missing"`; a broken note is a
  validation problem, not a migration target.
- `note_type=note.type` (already normalized to `core`/`reference`).
- `description=` stays the synced long description for reference notes; for core notes
  pass `note.description` so an existing description on a core note survives rather than
  being dropped.
- Rename `AmdLongMemoryDescriptionUpdate` → `AmdMemoryFrontmatterUpdate` in
  `src/sase/amd/_shared.py` and its two use sites in `root_rendering.py`, and update
  `AmdMemorySyncPlan.description_updates` → `frontmatter_updates`. Update the
  `MemoryExpectedFile` detail string to `"memory note frontmatter"`.

The net effect: `sase memory init` reports
`update sase/memory/<note>.md — memory note frontmatter` for every note still on a
legacy value, and converges in one pass. Because the update content also feeds
`note_overlay`, the regenerated `README.md` and `AGENTS.md` see the migrated values in
the same run.

Guard against churn: a core note whose frontmatter is already canonical must produce no
change. The existing `if content != text` check gives this, but it is exactly what the
new test in step 10 pins.

## Step 9 — Regenerate

Run **after** every source change, from the workspace, using the workspace venv so the
new code is what regenerates:

```bash
.venv/bin/sase memory init --check --diff   # review first
.venv/bin/sase memory init
.venv/bin/sase memory init --check          # must be clean
```

This writes, in this repo: `sase/memory/*.md` (frontmatter only),
`sase/memory/README.md`, `AGENTS.md`, and the four provider shims.

**It also writes the chezmoi home root** (`use_chezmoi: true`, home root
`~/.local/share/chezmoi/home`): `sase/memory/sase.md`, `sase/memory/obsidian.md`,
`sase/memory/README.md`, home `AGENTS.md`, and the home shims. That is
`sase memory init`'s designed behavior and there is no project-only flag. Two
consequences to handle, not to work around:

1. **Do not hand-edit anything under `~/.local/share/chezmoi/`.** Let `sase memory init`
   be the only writer. Reading or editing that repo any other way requires `/sase_repo`.
2. **Land ordering matters.** Until this change is on `sase` master and the primary
   checkout the globally installed `sase` runs from is updated, that old binary will
   read the migrated home notes as invalid-typed: `sase memory read obsidian.md` fails
   and home notes drop out of Tier 1/Tier 2 rendering. The exposure is a narrow,
   non-crashing window that closes at land. Call it out explicitly in the closing note
   so the land agent sequences `sase-core` release → `sase` master → chezmoi apply, and
   so the user is not surprised by a chezmoi diff.

Also add the five glossary terms in this same pass, before the final regeneration, so
there is exactly one regeneration (the epic plan's "vocabulary lock"):

```bash
.venv/bin/sase glossary add "Core Memory" "..." -I
.venv/bin/sase glossary add "Reference Memory" "..." -I
.venv/bin/sase glossary add "Memory Web" "..." -I
.venv/bin/sase glossary add "Memory Strand" "..." -I
.venv/bin/sase glossary add "Strand Keyword" "..." -I
```

Use `-I/--no-init` on each so only the final `sase memory init` regenerates. Definitions
to author (keep each to the two or three sentences the existing entries use, and let
them reference each other — `sase glossary add` validates cross-references through
Rust):

- **Core Memory** — a SASE memory that renders as always-loaded Tier 1 context, declared
  by `type: core` frontmatter and inlined into `AGENTS.md` and every provider
  instruction shim. Legacy `type: short` means the same thing. Alias: `core memory`.
- **Reference Memory** — a SASE memory that renders as Tier 2 detail, declared by
  `type: reference` frontmatter, named in `AGENTS.md` with a description, and fetched
  explicitly through an audited `sase memory read`. Legacy `type: long` means the same
  thing. Alias: `reference memory`.
- **Memory Web** — a memory note that names a keyed collection of small sibling notes.
  Not yet implemented; the term is reserved now so the vocabulary is stable. Define it
  as the descriptor, and say that a web _renders as_ core or reference memory rather
  than _being_ one.
- **Memory Strand** — one small note inside a memory web, never inlined into a generated
  document and read on demand by keyword. Not yet implemented.
- **Strand Keyword** — the display key a memory strand is addressed by, distinct from
  its filename slug, which is its identity. Not yet implemented.

Each of the last three must say plainly that it is not yet available, so an agent
reading the glossary does not try to use it.

## Step 10 — Tests

Existing tests carrying the old spellings must be updated, not deleted:
`tests/main/test_init_memory_agents_templates.py` (18 occurrences),
`tests/main/test_memory_agent_docs_list.py` (7), `tests/main/test_section_numbers.py`
(4), `tests/main/test_init_memory_managed_agents.py`, `test_init_memory_glossary.py`,
`test_init_memory_validation.py`, `test_init_memory_markdown_templates.py`,
`test_init_memory_agent_docs.py`, `test_init_onboarding_memory.py`,
`test_parser_root_help.py`, `test_parser_command_help.py`,
`tests/test_memory_inventory.py`, plus the ~25 files whose fixtures write `type: short`
/ `type: long` (`tests/memory/`, `tests/ace/tui/`,
`tests/test_xprompt_memory_loader.py`, `tests/test_xprompt_catalog_structured.py`,
`tests/_xprompt_catalog_helpers.py`, …).

Where a fixture is only setting up an unrelated scenario, move it to the new spelling.
Where a fixture is _about_ compatibility, keep the old spelling deliberately and say so
in the test name.

New tests — these are the phase's actual acceptance criteria:

1. `parse_memory_note_text` on `type: short` yields `type == "core"` and
   `type_source == "frontmatter"`; same for `long` → `reference`. `type: core` and
   `type: reference` yield themselves. `type: bogus` still yields
   `type_source == "invalid"` and preserves the raw value.
2. An `AGENTS.md` generated with the **old** anchors (`Tier 1 (short-term) Memory`,
   `Tier 2 (long-term) Memory`) still round-trips through `parse_amd_agents_document` —
   `has_short_section`, `has_long_section`, `short_memory_paths`, and
   `long_memory_entries` all populated. This is the regression that protects every
   unmigrated project.
3. A freshly rendered `AGENTS.md` emits the **new** anchors, and its numbered form
   (`## 1. Tier 1 (core) Memory`) still parses.
4. `render_markdown_template` with a README override that uses only `short_notes` and
   `long_notes` renders successfully and shows the right counts; an override using
   `core_notes`/`reference_notes` renders too; an override using a genuinely unknown
   variable still errors.
5. `sase memory init` on a fixture root containing a `type: short` core note and a
   `type: long` reference note plans exactly two `update` changes with the frontmatter
   detail, writes `type: core` / `type: reference`, preserves bodies and extra
   frontmatter keys byte-for-byte apart from the `type:` line, and reports **no**
   changes on a second run (idempotence).
6. A core note that already says `type: core` produces no change — the no-churn guard.
7. `memory_note_issue` via the Python binding accepts `core` and `reference` and still
   accepts `short` and `long`; a memory xprompt loaded from a `type: core` note gets
   `memory_type == "core"`.
8. `sase memory read` rejects a `type: core` note with the always-loaded message and
   accepts a `type: reference` note; a legacy `type: long` note is still readable.
9. The ACE memory-panel add modal offers `core`/`reference` and the delete modal renders
   `Tier 1 (core)` / `Tier 2 (reference)`.

## Step 11 — Verify

```bash
just install
just check
```

If ACE memory-panel PNG goldens fail
(`tests/ace/tui/visual/test_ace_png_snapshots_memory_panel.py`), the add/delete modal
label changes are the cause. Inspect `.pytest_cache/sase-visual/` and accept with
`just test-visual --sase-update-visual-snapshots` only after confirming the diff is
exactly the retermed labels.

This change touches the broadening set (`src/sase/memory/`, `src/sase/amd/`,
`src/sase/main/init_memory/`, templates, and the xprompt loader), so `just check`'s
scoped lane is not sufficient. Finish with a monitored full run:

```bash
sase monitor start --command 'just check-full' \
  --start-status TESTING --stop-status TESTED \
  --next 'Report just check-full results for sase-sq.1 and close the bead if green'
```

## Risks

| Risk                                                                              | Severity | Mitigation                                                                                                                    |
| --------------------------------------------------------------------------------- | -------- | ----------------------------------------------------------------------------------------------------------------------------- |
| A newer `sase` stops parsing already-generated `AGENTS.md` in unmigrated trees    | high     | Anchors are widened, never replaced; test 2 pins it                                                                           |
| An older `sase-core` rejects `type: core` notes                                   | high     | Step 1 lands and releases first; `just install` builds the local checkout so nothing here is verified against a stale binding |
| Migrated chezmoi home notes break the currently installed `sase`                  | medium   | Narrow, non-crashing window; called out for land sequencing in step 9 and in the closing note                                 |
| The README template rename breaks a user's overridden template                    | medium   | Both variable spellings are fed; `optional_variables` stops the unknown-placeholder error; test 4                             |
| Frontmatter rewriting churns notes it should not touch                            | medium   | Skip generated and invalid-typed notes; tests 5 and 6 pin idempotence and no-churn                                            |
| Mobile catalog payloads written by an older `sase` fail to deserialize            | medium   | `#[serde(alias = ...)]` on both variants; the legacy-payload deserialization test in 1.4                                      |
| The `sase-core-rs` floor is not bumped, so CI installs a wheel without the change | medium   | Explicit follow-up note; CI builds from the checkout, and the published floor bump lands with the release                     |
| Visual snapshots drift unnoticed                                                  | low      | Step 11 names the suite and requires inspecting the diff before accepting                                                     |

## Definition of done

- `./scripts/check.sh` is green in the `sase-core` checkout.
- The step 6 grep returns nothing outside the two excluded paths.
- `.venv/bin/sase memory init --check` is clean in this repo.
- `just check-full` is green.
- Five glossary terms exist and appear in the regenerated `AGENTS.md`.
- `PROPOSED FOLLOW-UP:` notes recorded on `sase-sq.1` for: the `sase-core-rs` floor bump
  after release, `bob-cli` memory regeneration, `sase skill init` redeployment of
  `/sase_memory_read`, the memory-directory-map image and prompt, and the deferred
  identifier rename.
