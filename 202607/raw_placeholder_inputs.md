---
tier: epic
title: Raw `<placeholder>` tags become prompt input arguments
goal: 'Submitting a prompt that contains raw `<placeholder>` tags opens one page that
  collects a value for each unique tag (alongside any declared `input:` arguments)
  and substitutes them before launch, and saving that draft as a global or local xprompt
  turns the same tags into `text` input arguments wired into the saved body.

  '
phases:
- id: core
  title: Raw-placeholder rules and transforms in sase-core
  depends_on: []
  size: medium
  description: '''Phase core'' section: add the raw/literal classification, raw-field
    summaries, span-safe substitution, and input-name slugging to sase-core plus their
    Python bindings.'
- id: facade
  title: Python facade and raw-only placeholder semantics
  depends_on:
  - core
  size: small
  description: '''Phase facade'' section: add the sase.xprompt.raw_placeholders facade
    and make highlighting, completion candidates, and common-placeholder recording
    raw-only.'
- id: plan
  title: Unified prompt input plan and placeholder substitution
  depends_on:
  - facade
  size: small
  description: '''Phase plan'' section: build the pure-logic PromptInputPlan that
    merges raw placeholders with declared frontmatter inputs and applies collected
    values to a prompt.'
- id: panel
  title: Prompt Inputs panel
  depends_on:
  - plan
  size: medium
  description: '''Phase panel'' section: grow InputCollectionModal into the single-page
    Prompt Inputs panel with a placeholder section, context snippets, keep-literal,
    and PNG snapshots.'
- id: submit
  title: Wire collection into the prompt-bar launch path
  depends_on:
  - panel
  size: small
  description: '''Phase submit'' section: gate the panel on submit from the ACE prompt
    bar, substitute values before launch, record tags, and add the config toggles.'
- id: xpromptargs
  title: Placeholders become xprompt input arguments
  depends_on:
  - facade
  size: medium
  description: '''Phase xpromptargs'' section: convert raw placeholders into text
    input arguments and Jinja references when saving a draft as a global (gx) or local
    (gX) xprompt.'
- id: docs
  title: Documentation, help popup, and end-to-end check
  depends_on:
  - submit
  - xpromptargs
  size: small
  description: '''Phase docs'' section: document the feature in ace.md/xprompt.md/configuration.md,
    update the ? help popup, and run the end-to-end verification checklist.'
create_time: 2026-07-26 06:06:46
status: done
bead_id: sase-9q
---

- **BEAD:** [sase-9q](https://github.com/sase-org/sase--beads/blob/main/pages/sase-9q/README.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-9q.land](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-9q.land/README.md)
  - [bbugyi200.athena.sase-9q.land--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-9q.land.md#member-code)
- **COMMITS:**
  - [f5f30f9](https://github.com/sase-org/sase/commit/f5f30f91e6f5c76b02d58b371d64761910448e39) — feat(ace): honor xprompt placeholder argument toggle (sase-9q)

# Plan: Raw `<placeholder>` tags become prompt input arguments

## Goal

Today a `<foobar>` tag in the ACE prompt bar is decoration: it is syntax-highlighted, it feeds the `<` completion menu,
and it is learned into the durable common-placeholder store — but it launches verbatim, so an agent receives a prompt
with a literal hole in it.

After this epic, a raw `<foobar>` behaves like a prompt input argument:

1. **On submit**, every unique raw placeholder in the draft is collected on one page, together with any `input:`
   arguments the prompt's frontmatter already declares, and the values are substituted into the prompt before launch.
2. **On xprompt save** (`gx` global / `gX` local), each unique raw placeholder becomes a `text` input argument of the
   new xprompt, and the saved body references it as `{{ name }}` so the argument is actually wired up.

## Terminology (used throughout this plan)

- **Placeholder span** — a complete, valid `<...>` occurrence as sase-core already defines it in
  `crates/sase_core/src/editor/placeholder.rs`: single line, non-empty inner text, no leading/trailing whitespace in the
  inner text, at most `PLACEHOLDER_MAX_INNER_CHARS` (100) inner characters.
- **Raw placeholder** — a placeholder span that does **not** intersect a launch-inert literal zone: an inline backtick
  span, a fenced code block, or a `%xprompts_enabled:false` disabled region. This is exactly the rule that already makes
  `#xprompt` references and `%directives` inert, so users need no new mental model.
- **Literal placeholder** — a placeholder span inside one of those zones. `` `<foobar>` `` is the escape hatch, and it
  is the one the user already reaches for.

## Key design decisions

Read these before implementing a phase; several phases would otherwise reinvent them differently.

**D1 — Backticks are the only escape hatch, and they are the existing one.** Rather than invent a new syntax (`\<foo>`,
`<<foo>>`, …), a raw placeholder is defined by the same literal-zone rule the launch parser already applies to
`#xprompt` references and `%directives`. sase-core already computes this composition in
`agent_launch::launch_literal_zone_ranges` and Python already mirrors it in
`src/sase/xprompt/_literal_zones.py::literal_zone_ranges`. Reusing it means `` `<div>` `` is inert for the same reason
`` `#foo` `` is.

**D2 — Highlighted means collected.** The prompt text area currently accents _every_ placeholder span, including ones
inside backticks and code fences. That becomes a lie the moment submission starts prompting for them. So the placeholder
overlay is narrowed to raw spans only; literal ones keep their inline-code / fenced-block styling. This makes the rule
self-teaching: if a tag is accented in the prompt bar, you will be asked for it.

**D3 — One page, one collection point.** A prompt can have both raw placeholders and declared `input:` arguments. Two
sequential modals would be miserable, so the existing `InputCollectionModal` grows a placeholder section and becomes the
single collection surface. Placeholders come first (they are positional in the user's text), declared inputs second
(required, then the existing collapsed optional group).

**D4 — Empty is not "blank", it is "keep literal".** A placeholder row is required by default: the confirm button stays
disabled and a status line reads `2 of 3 filled`. Launching an agent with an unfilled hole wastes a real agent run, so
silence must not be the default outcome. The escape is one discoverable key: `<ctrl+l>` toggles **keep literal** on the
focused placeholder row (the row dims, shows a `literal` chip, and counts as filled); pressing it with focus outside the
field list marks every still-empty placeholder row literal. A literal-marked placeholder is simply omitted from the
substitution map, so the original `<foobar>` text survives into the launched prompt.

**D5 — Substitution is span-based, single pass, never rescanned.** Values are inserted by byte range, not by
`str.replace`. This is what makes literal placeholders survive verbatim, and it guarantees that a value which itself
contains `<bar>` is inserted as text rather than becoming a new placeholder.

**D6 — Placeholders are substituted before declared-input Jinja rendering.** The order is: collect once → substitute raw
placeholders into the body → run the existing `render_prompt_with_inputs`. A prompt with no declared `input:` block runs
no Jinja at all, so placeholder values are perfectly literal in the common case. Document the interaction for the rare
prompt that has both.

**D7 — Prompt history records what actually ran.** The history entry stores the substituted prompt, matching how
declared inputs already behave and keeping history honest about what the agent saw. The _tags_ are still learned: the
collection path calls `record_prompt_placeholders` on the pre-substitution body at confirm time, so the common
placeholder store keeps growing. (Recovering the template itself is already served by the prompt stash.)

**D8 — Placeholder→input-argument conversion applies to _new_ xprompts only.** `gx` (save as xprompt) and `gX` (save as
local xprompt) convert; `gw` (write a bound xprompt) does not. The user asked for new xprompts, and a bound write
interacts with `PromptStackState.markdown_preserving_unchanged_body`, which deliberately preserves an untouched body
byte-for-byte. Rewriting there would silently defeat that guarantee.

**D9 — TUI-only, prompt-mode only.** Collection happens on the ACE prompt bar's agent-launch path
(`_finish_agent_launch`). Non-interactive launches (`sase run`, workflows, gates) are untouched: they cannot prompt, and
CLI prompts containing `<...>` are common and must not start failing. Feedback and approve-prompt bars are also out of
scope (they do not launch agents through this path).

**D10 — Xprompt bodies are not scanned.** Only the draft as typed is scanned. A `<foo>` living inside an xprompt body
that `#foo` will expand at launch time is not collected — expanding every reference before submit would be slow and
would surprise the user with tags they never wrote.

**D11 — Frontmatter is not scanned.** The scan and substitution run on the prompt _body_ only. Frontmatter is
configuration; a `description:` mentioning `<name>` must not open a collection page.

## Phase core

**Repo: `sase-core` (linked). Open it with `/sase_repo` (`sase repo open sase-core -r "<reason>"`) and use the printed
path as the only path for reads and writes.** Per the `rust_core_backend_boundary` memory, the classification rule, the
substitution transform, and the input-name slugging are shared backend behavior: the xprompt LSP and any future web or
CLI prompt composer must agree with the TUI exactly. All four live in Rust.

### core.1 — Expose the literal-zone composition

`crates/sase_core/src/agent_launch/mod.rs` already has private
`launch_literal_zone_ranges(prompt) -> Vec<(usize, usize)>` (fenced blocks + disabled regions + inline code masked by
directives/references/alt-directives). Make that composition reachable from `crates/sase_core/src/editor/`:

- Preferred: mark it `pub(crate)` and re-export a documented public
  `prompt_literal_zone_ranges(text: &str) -> Vec<(usize, usize)>` from `crates/sase_core/src/lib.rs`.
- If the `editor → agent_launch` direction proves awkward, extract the composition into a new `literal_zones` module
  that both `agent_launch` and `editor` depend on. Do **not** duplicate the logic; a second copy is a guaranteed drift
  bug.

Returned ranges are UTF-8 byte offsets, may be unsorted and overlapping (the existing callers tolerate that) — normalize
inside the new consumers rather than changing the producer's contract.

### core.2 — Classify spans as raw or literal

In `crates/sase_core/src/editor/placeholder.rs`:

- Add `pub raw: bool` to `PlaceholderSpan` (serialized, so it reaches Python and the LSP). `raw` is `true` when the
  span's full byte range does not intersect any literal zone.
- `extract_placeholder_spans` keeps returning **every** span, now tagged. This stays additive so the LSP and existing
  callers do not change shape.
- Narrow `build_placeholder_completion_candidates` to raw spans only: a `<foo>` written inside a code fence is a literal
  example, not a tag worth re-offering in the `<` menu. Update the existing
  `extracts_strict_single_line_spans_including_code` test — it currently asserts that `` `<inline>` `` and a fenced
  `<code value>` are extracted; they still are, now with `raw: false`.

### core.3 — Raw field summaries

Add to the same module:

```rust
pub struct RawPlaceholderField {
    pub text: String,       // inner text, e.g. "the plan"
    pub occurrences: usize, // raw occurrences of this exact inner text
    pub context: String,    // one-line snippet around the first raw occurrence
}

pub fn raw_placeholder_fields(text: &str, context_width: usize) -> Vec<RawPlaceholderField>;
```

- Ordered by first raw occurrence; deduped by **exact** inner text (`<Foo>` and `<foo>` are two fields — the collection
  page substitutes exact spans, so no case folding is safe here).
- `context` is the single line containing the first occurrence, trimmed and centered on the placeholder, ellipsized with
  `…` on either side to at most `context_width` characters, with the `<...>` text itself kept intact so the caller can
  style it. Returning the snippet from core (rather than raw offsets) keeps Python free of UTF-16/byte/char offset
  plumbing and keeps every frontend's snippet identical.
- Literal spans contribute neither a field nor an occurrence count.

### core.4 — Span-safe substitution

```rust
pub fn substitute_raw_placeholders(text: &str, values: &BTreeMap<String, String>) -> String;
```

- Rebuilds the string from raw span byte ranges, left to right, in one pass. A raw span whose inner text is a key in
  `values` is replaced (brackets included) by the mapped value; every other byte is copied verbatim.
- Literal spans are never touched. Keys with no matching span are ignored. **Inserted values are never rescanned**, so a
  value containing `<bar>` lands as text (D5).
- Tests must cover at minimum: replacement leaves `` `<x>` `` and a fenced `<x>` untouched while replacing the raw
  sibling on the same line; a value containing `<y>` is not re-substituted; a partially-populated map leaves the
  unmapped placeholders verbatim; empty map is the identity; multi-byte characters before a span do not shift output.

### core.5 — Input-name slugging

```rust
pub fn placeholder_input_names(texts: Vec<String>) -> Vec<String>;
```

Deterministic, order-preserving, one output per input. The rule:

1. Lowercase.
2. Replace every run of non-alphanumeric characters (`char::is_alphanumeric`, so Unicode letters survive — Jinja2 uses
   Python identifier rules) with `_`.
3. Strip leading and trailing `_`.
4. Truncate to 40 characters, then re-strip a trailing `_`.
5. If the result is empty, or begins with a numeric character, prefix `arg_`. If still empty, use `arg`.
6. Resolve collisions **among the produced names only** by appending `_2`, `_3`, … (so `<the plan>` and `<the-plan>`
   yield `the_plan` and `the_plan_2`). Do not consult any external reserved set — merging against a prompt's existing
   declared inputs is frontmatter policy and belongs in Python (see phase `xpromptargs`).

Examples to pin in tests: `<the plan>` → `the_plan`; `<PR #>` → `pr`; `<2fa code>` → `arg_2fa_code`; `<???>` → `arg`;
`<código>` → `código`.

### core.6 — Python bindings

In `crates/sase_core_py/src/lib.rs`, alongside the existing `py_placeholder_spans` / `py_placeholder_completion`:

- `raw_placeholder_fields(text: str, context_width: int) -> list[dict]`
- `substitute_raw_placeholders(text: str, values: dict[str, str]) -> str`
- `placeholder_input_names(texts: list[str]) -> list[str]`

Register all three in the module init, extend the module docstring binding list (around `lib.rs:152`), and add binding
smoke tests next to the existing `py_placeholder_spans` test (~`lib.rs:6816`) asserting the new `raw` key is present and
correct for ``"`<inline>` <live>"``.

### core.7 — Verification

Run the crate's own test suite in the sase-core checkout. In the sase repo, `just install` rebuilds `sase_core_rs` from
the linked checkout (`sase/repos/linked/sase-core`), so downstream phases pick the new bindings up automatically — call
that out in the phase's completion notes so the next agent runs `just install` first.

## Phase facade

**Repo: `sase` (this workspace).** Surface the new core behavior in Python and make the three existing placeholder
consumers raw-only.

### facade.1 — Typed facade

- Extend `PlaceholderSpan` in `src/sase/xprompt/placeholder_completion.py` with `raw: bool`, rehydrated in
  `_span_from_dict` with a tolerant default (`bool(payload.get("raw", True))`) so an older wheel degrades to today's
  behavior instead of raising.
- Add `src/sase/xprompt/raw_placeholders.py` — the typed facade for the three new bindings, following the existing
  `placeholder_completion.py` shape (`require_rust_binding`, frozen slotted dataclasses, no untyped dicts escaping):

  ```python
  DEFAULT_CONTEXT_WIDTH = 60

  @dataclass(frozen=True, slots=True)
  class RawPlaceholderField:
      text: str
      occurrences: int
      context: str

  def raw_placeholder_fields(text: str, context_width: int = DEFAULT_CONTEXT_WIDTH) -> tuple[RawPlaceholderField, ...]
  def substitute_raw_placeholders(text: str, values: Mapping[str, str]) -> str
  def placeholder_input_names(texts: Sequence[str]) -> tuple[str, ...]
  ```

- Tests in `tests/xprompt/test_raw_placeholders.py`, mirroring the monkeypatched-binding style of
  `tests/test_xprompt_placeholder_completion.py` plus a handful of real-binding round trips.

### facade.2 — Highlighting is raw-only (D2)

In `src/sase/ace/tui/widgets/_placeholder_highlight.py::_build_highlight_map`, skip spans with `raw is False` so the
inline-code and fenced-block styling from `_codeblock_syntax_highlight.py` shows through unmodified. Leave the
per-document scan cache, the `_MAX_OVERLAY_BYTES` / `_MAX_OVERLAY_LINES` guards, and the completion memo untouched.

### facade.3 — Common-placeholder store learns raw tags only

In `src/sase/history/prompt_placeholders.py::_placeholder_texts`, filter to raw spans. A tag the user only ever wrote
inside a code fence is documentation, not a tag they want offered back. Existing tests in
`tests/history/test_prompt_placeholders.py` use plain text and keep passing; add a case proving `` write `<alpha>` ``
records nothing while `write <alpha>` still records `alpha`.

### facade.4 — Visual snapshots

Refresh the affected goldens under `tests/ace/tui/visual/snapshots/png/` and add a case to
`tests/ace/tui/visual/test_ace_png_snapshots_prompt_highlighting.py` whose prompt contains a raw placeholder, a
backticked one, and a fenced one on adjacent lines — the accent must appear on exactly one of the three. Accept
intentional changes with `--sase-update-visual-snapshots`.

## Phase plan

**Repo: `sase`.** Pure logic, no Textual imports, mirroring `src/sase/agent/prompt_inputs.py`.

Add `src/sase/agent/prompt_placeholder_inputs.py` (named to avoid colliding with the unrelated
`sase.history.prompt_placeholders` store):

```python
@dataclass(frozen=True)
class PromptInputPlan:
    """Everything a submitted prompt needs collected, in display order."""
    placeholders: tuple[RawPlaceholderField, ...]
    declared: PromptInputRequest | None

    @property
    def needs_collection(self) -> bool: ...   # any placeholder, or a required declared input

@dataclass(frozen=True)
class PromptInputValues:
    placeholders: dict[str, str]   # exact inner text -> value; literal-marked rows omitted
    declared: dict[str, str]       # existing raw-string contract of InputCollectionModal

def build_prompt_input_plan(prompt: str) -> PromptInputPlan
def apply_prompt_input_values(prompt: str, values: PromptInputValues) -> str
```

- `build_prompt_input_plan` splits frontmatter with `parse_yaml_front_matter` (D11), runs `raw_placeholder_fields` on
  the body, and reuses `parse_prompt_input_request` for the declared block.
- `apply_prompt_input_values` implements D6: split frontmatter → `substitute_raw_placeholders(body, ...)` → reattach →
  `render_prompt_with_inputs(prompt, values.declared)`. It re-raises `PromptInputError` unchanged so callers keep one
  error path.
- Multi-agent drafts: the body is scanned and substituted as one string across `---` segments, so a tag used in two
  segments is collected once and applied to both — the same contract declared inputs already have.

Tests in `tests/agent/test_prompt_placeholder_inputs.py`: frontmatter excluded from the scan; placeholder + declared
input in one prompt; literal-marked placeholder survives; a value containing `{{ x }}` in a prompt with **no** declared
inputs is not rendered; multi-segment prompts.

## Phase panel

**Repo: `sase`.** Grow `src/sase/ace/tui/modals/input_collection_modal.py` into the single-page Prompt Inputs panel
(D3). Keep the class name `InputCollectionModal` — renaming would ripple through `modals/__init__.py`, `styles.tcss`,
and the visual suite for no user-visible gain — but retitle the surface and update the module docstring.

### panel.1 — Structure

- Constructor takes a `PromptInputPlan` (plus the existing `agent_count`) instead of a bare `PromptInputRequest`.
  Dismisses with `PromptInputValues` or `None`.
- Field indexing stays a single flat list so `_index_of`, `_field_value`, `_focus_next_field`, and the `#field-input-0`
  autofocus keep working: **placeholders first** (`0 .. P-1`), then required declared inputs, then optional ones.
- Section headers (`PLACEHOLDERS`, `INPUTS`) render only when both kinds are present.
- Title: `Fill in this prompt`; subtitle counts what is being collected, e.g. `3 placeholders · 1 input`.

### panel.2 — A placeholder row

```
<the plan>                                                    ×2
…implement <the plan> and report back…
[ ____________________________________________________________ ]
```

- The header renders the tag with the _same_ styling the prompt bar uses (`placeholder.delimiter` = accent dim,
  `placeholder.inner` = secondary bold) so the row is visually the same object the user just typed. Pull the colors from
  `self.app.current_theme` exactly as `_placeholder_highlight._register_placeholder_text_area_theme` does.
- The `×N` occurrence chip appears only when `occurrences > 1`.
- The context line is `RawPlaceholderField.context`, dim, with the tag itself accented inside it. This is the detail
  that makes the page feel considered: you see _where_ the hole is without leaving the panel.
- The editor is the existing `SingleLineVimTextArea` (`_InputCollectionInput`), so vim motions, `<enter>`-to-next-field,
  and the modal's live-validation plumbing all come for free.

### panel.3 — Keep literal (D4)

- `<ctrl+l>` toggles keep-literal on the focused placeholder row; with focus outside the field list it marks every
  still-empty placeholder row literal.
- A literal row dims, replaces its editor with a `literal · will stay as <the plan>` line, and counts as filled.
- Status line above the buttons reads `2 of 3 filled` (or `all filled`), and the confirm button stays disabled until
  every placeholder row is filled-or-literal **and** every required declared input validates. Declared inputs keep their
  existing `validate_and_convert` behavior and inline core guidance untouched.
- Footer hint line: `<enter> next field · ^l keep literal · <esc> cancel`.

### panel.4 — Styles and snapshots

- Extend the `InputCollectionModal` rules in `src/sase/ace/tui/styles.tcss` (~line 2508) with the new section headers,
  placeholder header, context line, literal chip, and status line. Match the surrounding modal conventions rather than
  inventing new spacing.
- Add PNG snapshot coverage in a new `tests/ace/tui/visual/test_ace_png_snapshots_prompt_inputs.py`, following
  `test_ace_png_snapshots_inputs.py`: (a) placeholders only, (b) placeholders + declared inputs with one row marked
  literal. Keep the existing two `input_collection_modal_*` goldens passing by keeping the declared-input-only rendering
  byte-identical where possible; regenerate them if the retitle changes them.
- Add non-visual widget tests for the keep-literal toggle, the filled counter, and the dismissal payload.

## Phase submit

**Repo: `sase`.** Wire the panel into the launch path and add the config toggles.

### submit.1 — Launch path

In `src/sase/ace/tui/actions/agent_workflow/_launch_start.py`:

- `_finish_agent_launch` replaces its `parse_prompt_input_request` gate with `build_prompt_input_plan`.
- When `plan.needs_collection` is false, take the existing fast path (including the current "optional declared inputs
  only → substitute defaults, no modal" branch).
- Otherwise call the renamed `_collect_prompt_inputs_then_launch(prompt, plan, keep_bar)`, which pushes the panel and,
  on confirm, calls `record_prompt_placeholders(body)` on the **pre-substitution** body (D7), then
  `apply_prompt_input_values`, then `_launch_resolved_prompt`.
- Cancel keeps today's behavior exactly: the prompt bar stays mounted with the draft intact and nothing launches.
- `agent_count` for the confirm label keeps coming from `parse_multi_prompt(prompt).segments`.

Everything downstream of `_launch_resolved_prompt` is untouched, so history, stash recovery, and the failed-launch path
keep working on the resolved prompt.

### submit.2 — Config

Add a new `ace.prompt_inputs` section (both keys default `true`):

- `src/sase/default_config.yml`: `collect_raw_placeholders` (submit-time collection) and `xprompt_placeholder_args`
  (phase `xpromptargs` conversion).
- `src/sase/config/sase.schema.json`: matching properties with descriptions and defaults, alongside the existing
  `ace.prompt_completion` block.
- Read them through the cached merged config (`load_merged_config`) the way
  `prompt_placeholders._common_placeholder_limit` does, so the submit path adds no disk I/O. With
  `collect_raw_placeholders: false`, `build_prompt_input_plan` returns no placeholder fields and behavior is exactly
  today's.

### submit.3 — Tests

Extend the ACE TUI action tests: submitting a draft with raw placeholders pushes the panel; confirming substitutes and
launches the resolved prompt; cancelling launches nothing and leaves the bar mounted; a draft whose only placeholders
are backticked launches immediately; the config toggle disables collection.

## Phase xpromptargs

**Repo: `sase`.** Turn raw placeholders into `text` input arguments when a draft becomes a _new_ xprompt (D8). Depends
only on `facade`, so it can run in parallel with `plan`/`panel`/`submit`.

### xpromptargs.1 — Shared conversion helper

Add to `src/sase/ace/tui/widgets/_local_xprompt_conversion.py` (or a sibling module if that file grows past its current
scope) a pure helper:

```python
@dataclass(frozen=True)
class PlaceholderArgConversion:
    body: str                  # body with raw placeholders rewritten to {{ name }}
    inputs: list[InputArg]     # one TEXT InputArg per newly introduced name, in document order
    renames: dict[str, str]    # placeholder inner text -> chosen input name (for messaging/preview)

def convert_placeholders_to_inputs(body: str, *, existing: Iterable[str] = ()) -> PlaceholderArgConversion
```

- Names come from `placeholder_input_names` (core.5).
- **Reuse, do not collide, against `existing`**: when a produced name already exists — as a declared `input:` in the
  draft's frontmatter, or as an undeclared Jinja variable in the body — the placeholder binds to that name and **no new
  `InputArg` is emitted**, so an existing declaration keeps its type, default, and description. This is the intuitive
  outcome for a draft that says both `<service>` and `{{ service }}`.
- The rewrite itself is `substitute_raw_placeholders(body, {text: "{{ name }}" ...})` (core.4), so literal placeholders
  in code fences are preserved verbatim in the saved xprompt — exactly what someone documenting placeholder syntax
  wants.
- New `InputArg`s are `InputType.TEXT` with no default (required), per the user's request; the user can retype them
  afterwards in the Frontmatter Panel.

### xpromptargs.2 — `gx`, save as a global xprompt

In `src/sase/ace/tui/actions/agent_workflow/_prompt_bar_save_xprompt.py`:

- After `_captured_xprompt_body` / `_captured_xprompt_frontmatter`, run the conversion with `existing` = declared input
  names from the captured frontmatter ∪ `inspect_template(body).unknown_variables`.
- Merge the produced `InputArg`s into the frontmatter with `PromptFrontmatter.set_input` (which replaces in place by
  name, so a pre-existing declaration is never clobbered), and hand the rewritten body to `UnifiedXPromptSaveModal`.
- The modal's existing draft-preview pane then shows the rewritten body and the new `input:` block **before** anything
  is written, which is the review step that keeps this transform honest.
- Snippet mode is untouched: snippets have no frontmatter and their own `$1` tabstop model, so their body is saved
  verbatim.
- `gw` / `WriteXpromptRequested` is explicitly untouched (D8).

### xpromptargs.3 — `gX`, save as a local xprompt

In `src/sase/ace/tui/widgets/_local_xprompt_conversion.py` and
`src/sase/ace/tui/widgets/_prompt_input_bar_local_xprompt_actions.py`:

- `infer_local_xprompt_inputs(body)` today returns one `TEXT` `InputArg` per undeclared Jinja variable, or `None` on
  invalid Jinja. Extend it to also return the rewritten body: keep the `None` failure contract for invalid Jinja, and
  return `PlaceholderArgConversion`-shaped data otherwise, with placeholder-derived inputs appended after the Jinja ones
  (document order within each group).
- `convert_active_pane_to_local_xprompt` builds the local `XPrompt` from the **rewritten** body, so
  `local_xprompt_invocation_skeleton` produces `#_name(the_plan=$1, target_file=$2)$0` and each placeholder becomes a
  snippet tabstop in the rewritten pane. This is the payoff: a rough draft with three holes becomes a reusable local
  helper plus a call site whose cursor is already on the first hole.
- Preserve the existing behavior for a pane with neither Jinja nor placeholders (bare `#_name`, `target_mode` honored).

### xpromptargs.4 — Tests

Unit tests for `convert_placeholders_to_inputs` (name reuse against existing declarations, collision suffixes, literal
placeholders preserved, invalid Jinja still rejected) and widget/action tests for both `gx` and `gX` asserting the
frontmatter gains `input:` entries of type `text` and the body/skeleton reference them.

## Phase docs

**Repo: `sase`.** Documentation, the in-app help popup, and the end-to-end pass.

### docs.1 — User documentation

- `docs/ace.md`: a new subsection in the prompt-bar area near the existing placeholder-completion text (~line 2780)
  covering the submit-time collection page, the backtick escape hatch, keep-literal, and the two config keys. Update the
  `gx` / `gX` descriptions to mention the input-argument conversion.
- `docs/xprompt.md`: note in the Typed Inputs area that saving a draft with raw `<tags>` mints `text` inputs.
- `docs/configuration.md`: document `ace.prompt_inputs.collect_raw_placeholders` and
  `ace.prompt_inputs.xprompt_placeholder_args`.

### docs.2 — Help popup

Per `src/sase/ace/CLAUDE.md`, the `?` popup must stay in sync with `sase ace` behavior. Update
`src/sase/ace/tui/modals/help_modal/binding_common.py` (which currently lists only
`("<...>", "Saved placeholder completion")`) and any prompt-bar binding section so the collection page and `<ctrl+l>`
are discoverable. Respect the 57-character box width rule.

### docs.3 — End-to-end verification

Run `just install` first (ephemeral workspaces may have stale dependencies, and this phase's predecessors changed
`sase-core`), then `just check`, then `just test-visual`. Manually exercise in `sase ace`:

1. Type a draft reading ``refactor <the plan> and keep `<div>` literal``, submit, confirm the page lists exactly one
   field, fill it, and verify the launched prompt substituted the tag and left `` `<div>` `` alone.
2. Repeat with `<ctrl+l>` on the single row and verify the prompt launches with `<the plan>` intact.
3. Type a draft with two distinct tags, press `<ctrl+g> x`, and verify the save modal's preview shows `{{ ... }}` in the
   body and two `text` inputs in the frontmatter.
4. Repeat with `<ctrl+g> X` and verify the pane becomes `#_name(a=…, b=…)` with the cursor on the first tabstop.

## Out of scope

- Non-interactive launch paths (`sase run`, workflows, gates) — D9.
- Feedback and approve-prompt bars — D9.
- Scanning xprompt bodies that expand at launch time — D10.
- `gw` bound-xprompt writes — D8.
- Converting placeholders into snippet tabstops when saving a pane as a _snippet_ (a plausible follow-up, but snippets
  have their own `$1` model and no frontmatter).
- Storing the un-substituted template in prompt history — D7.
