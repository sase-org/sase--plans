---
tier: tale
title: Rename the prompt input widget to Prompt Composer
goal:
  '"Prompt Composer" is the one canonical name for the TUI prompt-authoring widget
  across docs, user-visible TUI strings, and a new glossary strand whose aliases keep
  old names resolving.'
size: medium
decisions:
  page_path:
    ask:
      Should the reference page also move from reference/tui/prompt-input.md to
      reference/tui/prompt-composer.md?
    choices:
      keep: Keep the URL; retitle the page, its H1, and its nav label only
      move: Move the page with tools/docs_move and leave a redirect from the old URL
    default: keep
    why:
      The retitle carries the rename; moving the page churns inbound links for no reader
      benefit
    answer: keep
  property_panel_name:
    ask: Should the new glossary entry call the FrontmatterPanel the "property panel"?
    choices:
      property:
        Entry says "property panel" and notes the UI and docs call it the frontmatter
        panel
      frontmatter:
        Entry says "frontmatter panel", matching the border title and the docs
    default: property
    why:
      The research's pick; the user, empty state, and property picker modals all say
      "property"
    answer: property
  memory_glossary_prompt_composer:
    ask:
      Add the Prompt Composer glossary strand and republish the generated instructions?
    memory:
      - glossary
      - glossary:prompt-composer
    default: true
    requested:
      I would like to rename the prompt input widget to "prompt composer" (update any
      appropriate documentation) and add a new glossary term for "prompt composer"?
    answer: true
proposed_by: bbugyi200.athena.1fv
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-25 07:02:55
status: wip
---

# Rename the prompt input widget to "Prompt Composer"

## Goal

The TUI's bottom-docked prompt-authoring widget currently has several names: "prompt
input widget", "prompt bar", "prompt input bar", "prompt widget", and "prompt input".
This change makes **Prompt Composer** the one canonical name everywhere a user or agent
reads it:

1. Add a `Prompt Composer` glossary strand to the glossary memory web. Its aliases are
   `composer`, `prompt bar`, and `prompt input widget`.
2. Update the documentation and the user-visible TUI strings to say "prompt composer".

Background: the research report
`research:202611/prompt_composer_glossary_term/prompt_composer_glossary_term.md`
compared the candidate names and drafted the glossary entry. Read it with
`sase artifact read research:202611/prompt_composer_glossary_term/prompt_composer_glossary_term.md "<why>"`
if you need the reasoning. Do not open sidecar files directly.

## Vocabulary rules for the whole change

- In running prose, write **prompt composer** in lowercase. In page titles, headings,
  nav labels, and help-section titles, write **Prompt Composer**. Once the full term
  appears in a paragraph, later mentions may shorten it to "composer".
- Name the parts consistently:
  - the **prompt stack**, which is unchanged;
  - a **pane**, which is unchanged;
  - the frontmatter panel. Docs headings and prose about the panel stay as they are.
    Only the glossary entry's wording depends on `property_panel_name`.
- Rename a phrase only when it means the widget. Some phrases that look similar refer to
  different features and must **not** be renamed:
  - **Prompt Inputs** (`ace.prompt_inputs`, the post-submit "Fill in this prompt" panel,
    raw `<placeholder>` inputs). For example, `ace.prompt_inputs` is described as
    "Prompt input collection settings…". Only the trailing "for sase's TUI prompt input"
    changes there.
  - The CLI's "multi-prompt input" (`docs/reference/agents/agent-launches.md`).
  - The Prompts overlay, prompt history, the prompt stash, and the Agents-tab prompt
    panel (`AgentPromptPanel`).
- Do not rename these widget-specific phrases mechanically; reword them as follows:
  - "home prompt bar" → "home prompt composer"
  - "prompt input stack" → "prompt stack"
  - "the prompt-bar subtitle" → "the composer subtitle"
  - "Plan Feedback and Coder Prompt bars" → "Plan Feedback and Coder Prompt composers"

## Out of scope (do not change)

- **Code identifiers.** This includes `PromptInputBar`, every `_prompt_input_bar_*` and
  `_prompt_bar_*` module, `FrontmatterPanel`, the `#prompt-input-bar` widget id, CSS
  classes and TCSS selectors, `PROMPT_INPUT_SECTION`, test names, and perf span names.
  The glossary strand names `PromptInputBar` and `FrontmatterPanel` so readers can get
  from the term to the code.
- **Internal docstrings, comments, and log messages.** The exceptions are the widget
  docstrings named in step 6. The strand's aliases already make old prose like "prompt
  bar" resolve to the new term.
- `CHANGELOG.md` and `docs/blog/posts/**`. Both are dated history. Their deep links stay
  valid because steps 2–3 keep legacy anchors.
- `docs/_meta/splits/*.yml`. These are historical split manifests that record the
  section names at split time. No tool checks them against current headings.
- Demo data (`demos/**`, `src/sase/fakey/scenarios/*.yml`), keymaps and their
  descriptions, and every linked repo (sase-nvim, sase-core, …).

## Steps

### 1. Glossary strand

> [!decision] memory_glossary_prompt_composer Create the strand below and republish with
> `sase memory init`.

Use the `/sase_memory_write` skill. Create the new `glossary:prompt-composer` strand
(slug `prompt-composer` in the glossary web's strand directory). Keep the body accurate
to current `master`. You may tighten wording, but keep it under about 270 words, which
is the longest existing strand. Here is the draft for `property_panel_name = property`:

```markdown
---
keyword: Prompt Composer
aliases:
  - composer
  - prompt bar
  - prompt input widget
---

A prompt composer is sase's bottom-docked TUI prompt-authoring widget (`PromptInputBar`)
whose vim-mode panes draft agent prompts, Plan Feedback, Coder Prompt replies, and bound
macro or snippet definitions (`✎ <ref>` chip; `gw` writes back). In prompt mode it holds
a **prompt stack**: one or more panes in top-to-bottom launch order, each holding one
agent segment (a `#name` macro swarm stays one pane until launch). Loaded text splits
into panes at top-level `---`; a typed `---` stays literal and `g-` adds a pane.
`gj`/`gk` move focus and `gJ`/`gK` reorder; `Enter` opens the submit chooser and
`g<enter>` launches only the active pane. Pane order alone never sequences agents — a
bare `%wait` waits on the pane above. `Ctrl+S` stashes a pane and `gs` the whole stack
to the [[glossary:prompt-stash]]. At most one pinned auxiliary pane (snippet `gt`,
mini-macro `gx`) edits a definition and never launches.

Its **property panel** (`FrontmatterPanel`; the border title and docs call it the
frontmatter panel; `g=` toggles it) sits above the stack and edits the stack's shared
YAML frontmatter as structured fields (`name`, `description`, `input`, `macros`, …) with
an `a` property picker and an `R` raw-YAML view. Local `_helper` macros declared there
complete in every pane, and a single-pane launch re-attaches the shared frontmatter; on
a mini-macro pane the panel switches to that macro's own frontmatter scope.

Distinct from the read-only Agents-tab prompt panel, the Prompts overlay, the
post-submit Prompt Inputs panel, and external `$EDITOR` sessions. Plan Feedback and
Coder Prompt reuse the widget as a single pane without a stack or property panel.
```

> [!decision] property_panel_name = frontmatter Lead the second paragraph with "Its
> **frontmatter panel** (`FrontmatterPanel`, also called the property panel; `g=`
> toggles it)". Change the last sentence to say "without a stack or frontmatter panel".

Then run `sase memory init` to regenerate `AGENTS.md`, `CLAUDE.md`, and the memory
README. The roster gains `Prompt Composer (composer, prompt bar, prompt input widget);`
immediately before `Prompt Stash (stash);`. The pinned roster line in
`tests/_macro_terminology_docs.py` sits before the insertion point, so its wrapping does
not change.

### 2. Retitle the reference page

In `docs/reference/tui/prompt-input.md`:

- Frontmatter: set `title: Prompt composer`. Set `description:` to "Insert mode,
  alt-brace syntax, macro editing, search, and launch-target picking in the sase TUI
  prompt composer."
- Change the H1 to `# Prompt composer` and update the intro sentence.
- Change `## Prompt Input Widget` to `## Prompt Composer`. Put
  `<a id="prompt-input-widget"></a>` on the line before the heading so external deep
  links keep working. The page already uses this legacy-anchor pattern, for example
  `<a id="prompt-stacks"></a>`.
- Rewrite the first body sentence ("The prompt input is a multiline TextArea widget…")
  so that it introduces the prompt composer. Link to the generated glossary entry
  (`../glossary.md#prompt-composer`) once.
- In `mkdocs.yml`, change the nav label `Prompt input: reference/tui/prompt-input.md` to
  `Prompt composer: …`.
- Change the "See also" link label `[Prompt input](prompt-input.md)` to
  `[Prompt composer](prompt-input.md)` in:
  - `docs/reference/tui/index.md` (keep its trailing summary)
  - `prompt-modes.md`
  - `prompt-completion.md`
  - `prompt-stacks.md`
  - `snippets.md`
  - `prompts-overlay.md`
- Point every internal `#prompt-input-widget` link at `#prompt-composer` and update the
  link text:
  - `docs/reference/integrations/editor.md`
  - `docs/reference/macros/directive-syntax.md`
  - `docs/reference/macros/arguments.md`
  - the `prompt-input-widget:` anchor target in `docs/_meta/moves.yml`. The generated
    `docs/ace.md` stub keeps its old anchor id and label.
- In the `description:` frontmatter of `prompt-modes.md`, `prompt-stacks.md`,
  `prompt-completion.md`, and `prompt-completion-panels.md`, replace "prompt widget" or
  "prompt input widget" with "prompt composer". Make the same replacement in each page's
  matching intro sentence.

> [!decision] page_path = move First make the edits above, then run
> `python tools/docs_move reference/tui/prompt-input.md reference/tui/prompt-composer.md`
> from the format venv (`just fmt-docs` shows which interpreter). Check that the tool
> rewrote the inbound links, `mkdocs.yml`, and `docs/_meta/moves.yml`. Then change
> `tests/test_agent_tribe_terminology.py` to list the new path. Do not pass `--stub`;
> the redirect is enough.

### 3. Retitle the snippet-authoring section

In `docs/reference/tui/snippets.md`, change
`### Authoring a snippet from the prompt bar` to
`### Authoring a snippet from the prompt composer`. Put
`<a id="authoring-a-snippet-from-the-prompt-bar"></a>` on the line before it. Then
update the links and labels that point at it:

- `docs/reference/config/ace-prompts.md`
- `docs/reference/tui/prompt-stacks.md`
- `docs/reference/tui/prompt-input.md`
- its `docs/_meta/moves.yml` anchor target. The generated `docs/ace.md` stub label
  stays.

### 4. Docs prose sweep

Find the candidates:

```bash
rg -n -i "prompt[ -](bar|input bar|input widget|widget)\b|prompt input\b" docs \
  -g '*.md' -g '!docs/blog/**' -g '!docs/README.md' -g '!docs/llms.txt' \
  -g '!docs/reference/glossary.md' -g '!docs/reference/config/*.md' \
  -g '!docs/reference/tui/keymaps.md' -g '!docs/ace.md'
```

Apply the vocabulary rules to every hit. Hand-written pages with hits at planning time:

- `reference/tui/`:
  - `prompt-input.md`
  - `prompt-stacks.md`
  - `snippets.md`
  - `prompts-overlay.md`
  - `prompt-modes.md`
  - `prompt-completion.md`
  - `prompt-completion-panels.md`
  - `agents-tab-modes.md`
  - `tabs-and-panes.md`
  - `projects-tab.md`
  - `launch-control.md`
  - `global-controls.md`
  - `agents-tab.md`
  - `agents-main-deck.md`
- `reference/config/ace-prompts.md`
- `reference/integrations/`:
  - `editor.md`
  - `remote-dispatch.md`
- `reference/prompt-history.md`
- `reference/macros/`:
  - `templating.md`
  - `directive-syntax.md`
  - `fields.md`
  - `arguments.md`
  - `troubleshooting.md`
  - `directive-composition.md`
  - `multi-agent-prompts.md`
- `guides/`:
  - `run-agents-in-parallel.md`
  - `reuse-prompt-with-macros.md`
  - `find-and-replay-prompt.md`
- `contribute/`:
  - `perf-runbook.md`
  - `workspace-provider.md`

Reword only to rename the widget. Do not restructure pages or add new prose beyond the
one glossary link in step 2. Guide pages must stay within their word budgets.

### 5. Config schema descriptions

The source of `docs/reference/config/tui.md` is `src/sase/config/sase.schema.json`.
Update these widget-meaning descriptions:

- `ace.snippets` → "Snippet expansions for the prompt composer (trigger string to
  expansion text)"
- `ace.snippet_config_path` → change "written from the prompt bar" to "written from the
  prompt composer"
- `ace.prompt_completion` → "Live completion settings for sase's TUI prompt composer"
- `ace.prompt_spellcheck` → "Sticky misspelling highlight settings for sase's TUI prompt
  composer"
- `ace.prompt_spellcheck.highlight` → change "in every prompt input" to "in every prompt
  composer"
- `ace.prompt_submission` → "Prompt submission confirmation settings for sase's TUI
  prompt composer"
- `ace.prompt_inputs` → "Prompt input collection settings for sase's TUI prompt
  composer"

Then search the schema for any other description that means the widget:

```bash
rg -n -i "prompt (bar|input|widget)" src/sase/config/sase.schema.json
```

### 6. User-visible TUI strings and their tests

Change these strings, and update the tests that pin them:

| File                                                               | Current                                                                         | New                                                                                 |
| ------------------------------------------------------------------ | ------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------- |
| `src/sase/ace/tui/modals/help_modal/binding_common.py`             | help section `"Prompt Input"`                                                   | `"Prompt Composer"` (keep the constant name)                                        |
| `src/sase/ace/tui/widgets/agent_onboarding.py`                     | "open the prompt bar (pre-filled…" / "The prompt bar works from any tab."       | "open the prompt composer (pre-filled…" / "The prompt composer works from any tab." |
| `src/sase/ace/tui/modals/jump_action_modal.py`                     | "Load into prompt input" / "Stash this bar, then edit the macro here."          | "Load into prompt composer" / "Stash this draft, then edit the macro here."         |
| `src/sase/ace/tui/widgets/_prompt_jump.py`                         | "No prompt input bar is available" / "Prompt input bar cannot load this target" | "No prompt composer is available" / "The prompt composer cannot load this target"   |
| `src/sase/ace/tui/actions/_stranded_resolve_actions.py`            | "The prompt bar is unavailable"                                                 | "The prompt composer is unavailable"                                                |
| `src/sase/ace/tui/actions/agent_workflow/_launch_prompt_inputs.py` | "…the prompt bar was closed while collecting inputs."                           | "…the prompt composer was closed while collecting inputs."                          |
| `src/sase/ace/tui/actions/agents/_hijack_prompt_bar.py`            | "…the prompt bar was closed while submitting."                                  | "…the prompt composer was closed while submitting."                                 |
| `src/sase/ace/tui/actions/agent_workflow/_prompt_bar_submit.py`    | toast title "Prompt input cancelled"                                            | "Prompt cancelled" (it pairs with "Prompt pane cancelled")                          |

Tests that pin these strings at planning time:

- `tests/ace/tui/test_help_modal_filter.py` and
  `tests/ace/tui/test_help_modal_filter_model.py` (`"Prompt Input"`)
- `tests/ace/tui/widgets/test_agent_onboarding.py`
- `tests/ace/tui/test_prompt_bar_stack_submit_handlers.py`
  (`"Prompt input cancelled — saved to history"`)

To find any other pins, grep `tests/` for each old string. Check that the help-modal
filter still matches the section by its new name, for example when a user filters on
"composer".

Docstrings: update only the docstrings of the `prompt_input_bar.py` module and the
`PromptInputBar` class, plus the `frontmatter_panel.py` module/class docstring. They
should introduce the terms, for example "Prompt composer widget (glossary: Prompt
Composer) …". Leave every other docstring alone.

### 7. Regenerate, then verify

1. Run `sase memory init`. Do this after step 1, and again if you edit the strand later.
2. Run `just fix`. It regenerates:
   - `docs/reference/glossary.md`, which gains the `#prompt-composer` entry;
   - `docs/README.md` and `docs/llms.txt`, from the page descriptions;
   - `docs/reference/config/tui.md`, from the schema;
   - the `docs/ace.md` moves stubs.
3. Read the strand back:
   `sase memory read glossary:composer -r "verify the new strand resolves"`. Check that
   `glossary:prompt-bar` and `glossary:prompt-input-widget` resolve to the same strand
   (use `sase memory show` for these extra checks so they are not recorded as reads).
4. Run `just docs-lint`. Then run `just docs-check`, the strict MkDocs build with anchor
   validation, so you know every renamed anchor and legacy anchor resolves.
5. Run `sase tool run check`.
6. Update the PNG goldens whose rendered text changed, using a targeted run. It may need
   `/sase_monitor`:

   ```bash
   just fix-tui-screenshots -- \
     tests/ace/tui/visual/test_ace_png_snapshots_agents_onboarding.py \
     tests/ace/tui/visual/test_ace_png_snapshots_help_panel.py \
     tests/ace/tui/visual/test_ace_png_snapshots_jump_action.py
   ```

   Inspect the report and every updated golden. The only differences should be the
   renamed strings. The glossary-highlight goldens use a fixture catalog, so the new
   strand cannot change them. Do not run a full visual suite or `just check-full`.

## Acceptance criteria

- The step 4 `rg` sweep finds nothing that means the widget. Anything left must be
  Prompt Inputs, CLI multi-prompt input, or historical/out-of-scope text.
- The same sweep over `src/sase/config/sase.schema.json` and the user-visible strings in
  step 6 finds nothing that means the widget.
- `AGENTS.md`, `CLAUDE.md`, and `sase/memory/glossary.md` list
  `Prompt Composer (composer, prompt bar, prompt input widget)`.
  `docs/reference/glossary.md` has a `## Prompt Composer {#prompt-composer}` entry.
- These old links still resolve:
  - `reference/tui/prompt-input.md#prompt-input-widget`
  - `reference/tui/snippets.md#authoring-a-snippet-from-the-prompt-bar`

  With `page_path = move`, they resolve through the redirect. `just docs-check` passes.

- `sase tool run check` passes, and the targeted visual run ends `clean` or `applied`
  with the goldens inspected.
