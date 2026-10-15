---
tier: epic
title: 'sase.sh: a small Diátaxis-typed Guide in front of a generated Reference'
goal: 'A newcomer on sase.sh meets one walked path: a one-screen homepage, two tested
  tutorials, about eight concept pages, and about fifteen goal-titled guides (~30K
  words). These sit in front of a lookup-sized Reference generated from the code that
  owns each fact. A lint-enforced documentation contract keeps the site from growing
  back.

  '
decisions:
  pdf_scope:
    ask: Which handbook PDF should sase.sh publish after the Guide/Reference split?
    choices:
      curate: Same URL, Guide tier plus the launch essay; Reference stays web-only;
        much faster deploys
      retire: Drop the PDF; the handbook URL redirects home and the PDF toolchain
        is deleted
      full: Keep the full-corpus PDF; deploys stay ~9 min and the 22 MiB gate likely
        re-breaches
    default: curate
    why: The full corpus nears the asset cap, dominates deploy time, and has no Zensical
      path
    answer: curate
  install_command:
    ask: Should documented install commands carry a version floor so a broken PyPI
      resolve fails loudly?
    choices:
      floor: Docs say uv tool install "sase>=0.18"; a bad resolve errors instead of
        installing 0.1.0
      bare: Docs say uv tool install sase; right only once 0.18.0 is on PyPI and the
        smoke stays green
    default: floor
    why: Truthful whether or not 0.18.0 (sase-1c1.14) ships before the new front door
    answer: floor
  docs_routing_note:
    ask: Add a docs_writing.md reference memory note that routes feature agents' doc
      changes to the right tier?
    memory:
    - docs_writing.md
    default: true
    requested: I agree with all of the requirements recommended in that research file
      (but keep in mind that you are responsible for planning and splitting up this
      work appropriately).
    answer: true
  docs_pointer_notes:
    ask: Update the docs/ path pointers in four memory notes when the decompose phase
      moves those pages?
    memory:
    - tui_perf.md
    - dispatch.md
    - macros.md
    - cli_rules.md
    default: true
    corrects: Command plugins section in `docs/plugins.md`.
    answer: true
phases:
- id: nav-tabs
  title: Six tabs over the existing pages, site UX features, analytics hook
  depends_on: []
  size: small
  description: 'nav-tabs: turn on Material tabs, breadcrumbs, and footer navigation,
    regroup today''s pages into the six tabs without moving files, and add the cookie-less
    analytics hook.'
- id: links
  title: Strict link validation, URL contract test, and docs checks in the gates
  depends_on: []
  size: medium
  description: 'links: add MkDocs validation, a sase.sh URL and anchor contract test,
    fix the two 404 onboarding links, and run the docs build in Master Gate and in
    just check.'
- id: applicability
  title: Declare which version the docs describe and smoke the documented install
  depends_on: []
  size: small
  description: 'applicability: add a generated latest-release line to the site header,
    single-source the documented install command, and schedule a clean PyPI install
    smoke.'
- id: pdf-handbook
  title: Execute the handbook PDF scope decision
  depends_on: []
  size: small
  description: 'pdf-handbook: make the PDF build follow pdf_scope, selecting pages
    by directory so later phases need no PDF edits.'
- id: contract
  title: Documentation contract lint, templates, and the Writing docs page
  depends_on:
  - nav-tabs
  size: medium
  description: 'contract: add tools/check_docs (directory typing, frontmatter, word
    budgets with a shrinking legacy allowlist, Guide hygiene, test-pin and portability
    rules), page templates, and the Contribute hub and Writing docs page.'
- id: routing
  title: Change-to-docs routing rule for agents and a constrained refresh_docs chop
  depends_on:
  - contract
  size: small
  description: 'routing: add the docs_writing.md reference memory note and constrain
    the refresh_docs chop prompts to page types, budgets, and the Guide tier''s fix-only
    rule.'
- id: moves
  title: Page-move machinery, docs map, and llms.txt
  depends_on:
  - links
  - contract
  size: medium
  description: 'moves: add docs/_meta/moves.yml as the single source for redirects
    and anchor stubs, the tools/render_docs framework, tools/docs_split, and a generated
    docs map plus llms.txt; pilot by taking Instruction Inventory off the site.'
- id: glossary
  title: Generated glossary, design-decision roster, tooltips, and the Reference hub
  depends_on:
  - moves
  size: medium
  description: 'glossary: generate the glossary and design-decision pages from the
    memory webs, add hover tooltips for about 15 core terms, and create the reference/index.md
    hub.'
- id: start
  title: Get started tier with two tutorials
  depends_on:
  - moves
  - applicability
  size: medium
  description: 'start: write start/index, install, first-run, and first-change from
    getting_started.md, then move the old page and its draft blog duplicate behind
    redirects.'
- id: tutorial-tests
  title: Executable tutorial tests on fakey
  depends_on:
  - start
  size: medium
  description: 'tutorial-tests: run every marked shell block in start/ against fakey
    in a temporary HOME, and add the runner to the scheduled Full CI lane.'
- id: concepts-model
  title: How SASE works plus the agents, work-units, and landing concepts
  depends_on:
  - glossary
  size: medium
  description: 'concepts-model: write concepts/index.md (one diagram, seven nouns)
    plus agents, work-units, and landing concept hubs, and move architecture.md''s
    internals to Contribute.'
- id: concepts-system
  title: Prompts, memory, automation, and providers concepts
  depends_on:
  - concepts-model
  size: medium
  description: 'concepts-system: write the remaining four concept hubs, each ending
    in Do it and Look it up links.'
- id: guides-setup
  title: Guides hub plus the Set up and Fix problems guides
  depends_on:
  - start
  - concepts-model
  size: medium
  description: 'guides-setup: create guides/index.md and seven goal-titled guides,
    folding troubleshooting/* and agent_auto_restart.md into them.'
- id: guides-daily
  title: Daily work guides
  depends_on:
  - guides-setup
  - concepts-system
  size: medium
  description: 'guides-daily: write the five daily-work guides, extracting how-to
    narrative from ace.md, macros.md, prompt.md, and related pages.'
- id: guides-automate
  title: Plan and automate guides
  depends_on:
  - guides-setup
  - concepts-system
  size: medium
  description: 'guides-automate: write the three plan-and-automate guides from sdd.md,
    beads.md, axe.md, and mentors.md.'
- id: cli-registry
  title: Generated CLI reference
  depends_on:
  - glossary
  size: medium
  description: 'cli-registry: generate one reference page per command group from the
    argparse tree and replace cli.md''s hand-written tables with links.'
- id: config-registry
  title: Generated configuration reference
  depends_on:
  - glossary
  - guides-setup
  size: medium
  description: 'config-registry: generate one reference page per config key family
    from sase.schema.json and default_config.yml, and delete the configuration.md
    text it duplicates.'
- id: keymap-directives
  title: Generated TUI keymap, directive, and macro discovery-order references
  depends_on:
  - glossary
  - guides-daily
  size: medium
  description: 'keymap-directives: generate the TUI keymap tables, the directive catalog,
    and macro discovery order, deleting the hand-written rows in ace.md and macros.md
    (fixes sase-1la).'
- id: homepage
  title: One-screen homepage, front-door link retargeting, and reader-testing kit
  depends_on:
  - start
  - concepts-system
  - guides-setup
  - applicability
  size: medium
  description: 'homepage: rebuild index.md around one primary CTA, install, one demo,
    four capability cards, and four doors; retarget README and TUI onboarding links;
    add the observed-reader kit.'
- id: decompose
  title: Decompose the monoliths and empty the legacy allowlist
  depends_on:
  - routing
  - tutorial-tests
  - guides-automate
  - cli-registry
  - config-registry
  - keymap-directives
  - homepage
  size: xlarge
  description: 'decompose: author and run a sub-epic that moves every remaining root
    page into reference/ or contribute/ with move-only commits, redirects, and anchor
    stubs; strips history into upgrading.md; and updates memory pointers.'
proposed_by: bbugyi200.athena.research.4l.linker.w0
decided_by: reviewer
decided_via: tui
create_time: 2026-10-15 18:11:07
status: wip
bead_id: sase-1le
---

- **PROMPT:** [prompts/202610/sase_sh_docs_redesign.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/sase_sh_docs_redesign.md)
- **BEAD:** [sase-1le](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1le/README.md)

# sase.sh: a small Diátaxis-typed Guide in front of a generated Reference

sase.sh publishes a **518K-word behavior spec as a user manual**. Two pages (`ace.md` at
103K words and `configuration.md` at 71K) hold a third of it. The navigation sorts about
60 always-expanded links by the author's sense of difficulty ("The Basics", "The Nitty
Gritty"), not by what the reader needs. The corpus grew 37% in the last month, almost
entirely as a side effect of feature commits. Nothing says what a page is for, so pages
drift toward exhaustive reference, which is the easiest text for a feature agent to
write.

This epic implements the recommendation in
`research:202610/sase_sh_docs_redesign/sase_sh_docs_redesign.md`. Bryan accepted every
requirement in that report (R0–R9). The approach has three parts:

- **Simplify what readers meet.** The Guide (Get started, Guides, Concepts) is small,
  typed, budgeted, and curated. It launches at about 30K words, with 60K as a hard
  ceiling.
- **Shrink what is hand-written, without deleting the spec.** Registries (CLI, config,
  keymaps, directives, glossary) are generated from the code that owns them. The
  remaining prose is split into lookup-sized Reference pages. Agents and power users
  keep every fact.
- **Make the structure enforceable.** Directory-typed pages, templates, word budgets
  with a ratchet, Guide hygiene, a URL contract, and a change-to-docs routing rule all
  run in `just check`. Without them, a reorganized tree drifts back within weeks.

This is an epic because the work splits into independent tracks: site shell, gates,
release alignment, PDF, Guide writing, registry generation, and Reference decomposition.
The phases touch disjoint files wherever possible, so several agents can work in
parallel. The 400K-word Reference decomposition is one `xlarge` phase on purpose. Its
section-by-section mapping depends on the split tooling and on what the Guide phases
extract, so its worker plans it as a sub-epic once those exist.

## Target information architecture

```text
sase.sh
├─ Home                          one screen: what · who for · install · demo · four doors
├─ Get started   (tutorials)     ┐
├─ Guides        (how-to)        ├─ THE GUIDE: curated, typed, budgeted, human-reviewed
├─ Concepts      (explanation)   ┘
├─ Reference     (reference)     ── generated where it is a registry, split where it is prose
├─ Contribute    (contributor)   ── development, internals, contracts, runbooks, the docs process
└─ Blog          (essays)        ── dated; never the only home of an instruction
```

- **The directory is the type:** `docs/start/`, `docs/guides/`, `docs/concepts/`,
  `docs/reference/`, `docs/contribute/`, `docs/blog/`. The only root pages left at the
  end are `index.md` and generated compatibility stubs. Frontmatter holds only `title`,
  `description` (one line, required in typed directories), optional `search.boost`,
  `hide`, and, for reference pages, an optional `covers:` code-glob list.
- **Labels are user labels, not framework labels.** Diátaxis governs how pages are
  written, not what the tabs are called. The system is explained once, in the homepage's
  "How these docs are organized" doors: Learn → Get started, Do → Guides, Understand →
  Concepts, Look up → Reference.
- **Reader promises:** the Guide never sends a newcomer into the Reference to finish a
  task. The Reference never re-teaches; it links back to Guides and Concepts.
- **Concept pages are the subsystem hubs.** Each ends with "Do it" links (guides) and
  "Look it up" links (reference). There is no fifth page type and there are no empty
  portals.
- **New files use lowercase hyphenated names** (`start/first-run.md` →
  `/start/first-run/`). Each directory gets an `index.md` hub of at most 400 words.

Target navigation once the epic lands (first-wave pages only; more guides arrive only
when a real user goal appears, and a guide title must start with a verb and name an
outcome):

```yaml
nav:
  - Home: index.md
  - Get started:
      - start/index.md # Is SASE for you? alpha status, requirements, the two lessons
      - Install and set up a provider: start/install.md
      - Run your first agent: start/first-run.md
      - Make and review a change: start/first-change.md
  - Guides:
      - guides/index.md
      - Set up:
          [
            add an existing repository,
            choose providers and models,
            configure the essentials,
            set up completion and your editor,
          ]
      - Daily work:
          [
            run agents in parallel,
            review and land a change,
            hand a result to another agent,
            reuse a prompt with macros,
            find and replay an old prompt,
          ]
      - Plan and automate:
          [
            plan a multi-agent change,
            schedule recurring work,
            get automated code review,
          ]
      - Fix problems:
          [start with sase doctor, unstick a queued agent, recover an interrupted agent]
  - Concepts:
      - How SASE works: concepts/index.md
      - Agents, sessions, and workspaces: concepts/agents.md
      - Work units — Patches, beads, plans, goals: concepts/work-units.md
      - How changes land: concepts/landing.md
      - Prompts, macros, and workflows: concepts/prompts.md
      - Memory and agent instructions: concepts/memory.md
      - Automation and human checkpoints: concepts/automation.md
      - Providers and models: concepts/providers.md
  - Reference:
      - reference/index.md # routing index by product unit
      - CLI: reference/cli/ # generated, one page per command group
      - Configuration: reference/config/ # generated per key family + prose
      - TUI: reference/tui/ # one page per tab + generated keymap
      - Macros and workflows: reference/macros/ # syntax + generated directives
      - Work units, providers, memory, automation, file formats, environment variables
      - Glossary: reference/glossary.md # generated from the glossary web
      - Design decisions: reference/design-decisions.md # generated from the decisions web
      - Upgrading and deprecations: reference/upgrading.md # the only home for history
  - Contribute:
      - contribute/index.md # development, writing docs, architecture internals,
        # Rust boundary, pane contracts, perf, telemetry, fakey, runbooks
  - Blog: blog/index.md
```

### How this makes sase.sh more intuitive, by visitor

- **Evaluator:** reads one sentence, sees one demo, learns the requirements and alpha
  status, and clicks one button ("Run your first agent").
- **New user:** finishes a tested 15-minute lesson in the managed `+home` project with a
  run they can see, then is offered one next lesson, not a 20-term component map.
- **Daily user:** lives in the Guides and Reference tabs. A config key is one generated
  section page, not a 7,768-line scroll. Keymaps are one generated table.
- **Contributor or agent:** contracts and internals sit under Contribute. The full spec
  stays in Reference, and a generated docs map (`docs/README.md`, `/llms.txt`) lists
  every page with its one-line description.

## The documentation contract (summary)

The `contract` phase builds the lint and the `docs/contribute/writing-docs.md` page.
Every later phase follows them.

| Tier                            | Per-page words                     | Tier total | Notes                                    |
| ------------------------------- | ---------------------------------- | ---------- | ---------------------------------------- |
| `start/`                        | ≤ 2,500                            | ≤ 10K      | Tutorials: no options, alternatives, why |
| `guides/`                       | target ≤ 1,500, hard ≤ 2,500       | —          | Anything longer is two guides            |
| `concepts/`                     | ≤ 3,000                            | ≤ 25K      | One diagram allowed per page             |
| Hubs (`*/index.md`)             | ≤ 400                              | —          | Orientation sentence, grouped links      |
| `reference/` (hand-written)     | ≤ 6,000                            | —          | Generated pages and blocks are exempt    |
| `contribute/`                   | ≤ 8,000 (warning only)             | —          |                                          |
| Legacy root pages (allowlisted) | ceiling = size at lint landing +1% | —          | Ceilings only shrink; empty at epic end  |

Words are counted outside frontmatter, fenced code, HTML comments, and generated blocks.

- **Guide hygiene** (`start/`, `guides/`, `concepts/` only). These are forbidden:
  - bead IDs (`sase-…`);
  - `src/` paths;
  - epic or phase labels such as "E3" or "phase sase-1eq.6";
  - history phrases ("formerly", "no longer", "legacy", "renamed", "still mentions").

  History goes to the CHANGELOG, decision records, and `reference/upgrading.md`.

- **Templates** live in `docs/_templates/`, excluded from the build: tutorial, how-to,
  concept, reference, and hub. Each page opens with one sentence saying what it is for.
  Tutorials and guides add a "Before you start" line. Every page closes with a "Next" or
  "Related" link.
- **Titles:** Guide titles are goals ("Get automated code review", not "Mentors").
  Reference titles name the thing.
- **Tests do not pin Guide prose.** A test may not read files under `docs/start/`,
  `docs/guides/`, or `docs/concepts/`. The only exceptions are the tutorial runner and
  `check_docs`'s own tests.
- **Portability (Zensical).** Add no MkDocs `hooks:`, `gen-files`, or plugins beyond
  today's allowlist (search, redirects, blog, rss, and exporter in the PDF config only).
  Generation happens in `tools/` before the build, with `--check` drift detection, the
  same way `tools/render_model_docs` already works. Markdown extensions (abbr, snippets,
  superfences custom fences) are fine.

## URL compatibility policy

Inbound links are an interface. README links are frozen into every released PyPI long
description, and TUI onboarding widgets ship sase.sh URLs.

- **Source of truth:** `docs/_meta/moves.yml` (built by `moves`). It holds one entry per
  moved page (`from`, `to`) plus an optional `anchors:` map for anchors that moved to a
  different page.
- **Whole-page moves** become generated `docs/_redirects` 301 lines. Browsers carry the
  original `#fragment` across a redirect whose `Location` has none, so same-page anchors
  keep working.
- **Split pages** (anchors spread across several new pages) and **pages cited by path in
  text this epic may not edit** instead get a generated stub page at the old path. That
  text is the immutable decision records (`docs/tool.md`, `docs/macros.md`) and the core
  memory note `rust_core_backend_boundary.md` (`docs/rust_backend.md`). A stub lists
  where the content went, carries a small inline fragment → URL map, is excluded from
  search and nav, and is readable by agents grepping `docs/`. No `_redirects` rule may
  shadow a stub.
- **The contract test** (built by `links`) resolves every frozen URL and anchor in the
  built site, following `_redirects` and stub maps. The frozen set covers:
  - every page URL that exists today;
  - every `https://sase.sh/...` URL in `src/`, `README.md`, and the latest released
    README;
  - every `_redirects` target.

  A deep link may never be "kept compatible" by redirecting it to a generic section
  index.

## Rules for every phase

- **Context:** read this plan, then `docs/contribute/writing-docs.md` once `contract`
  has landed. Read `lint_and_test.md` through `/sase_memory_read`. Read `tui.md` only if
  you capture TUI screenshots. The research report is background, not a spec. This plan
  wins where the two differ.
- **New Guide content goes in new files.** Never bulk-reflow a legacy page.
- **Extract; don't copy.** When a Guide page takes how-to narrative from a legacy page,
  replace that passage with a link in the same commit. Facts (tables, defaults, flags)
  stay in the Reference, and the Guide links to them instead of restating them.
- **Moving content:** first a move-only commit (with `tools/docs_split` once `moves` has
  landed), then content edits in separate commits. Record every move in
  `docs/_meta/moves.yml`.
- **When a doc path moves,** update every literal reference in the same commit:
  - tests, including the terminology allowlists in `tests/_macro_terminology_docs.py`,
    `tests/test_agent_session_terminology.py`, `tests/test_agent_tribe_terminology.py`,
    and `tests/test_plugins_required_gate_preview.py`;
  - `src/` help text and comments;
  - `README.md`;
  - `mkdocs.yml` and `mkdocs-pdf.yml`;
  - links in other pages.

  Grep for the old path before committing. Memory notes change only where an accepted
  memory decision covers them.

- **Hot pages** (`ace.md`, `configuration.md`, `macros.md`, `beads.md`, `llms.md`) take
  about 9 agent commits a day. Rebase right before editing one, keep each commit small
  and single-purpose, and land promptly.
- **Nav edits** stay inside your own tab or sub-group in `mkdocs.yml`. On a rebase
  conflict, keep both sides.
- **Accuracy first:**
  - Verify every command, flag, and output you document against current master: run
    `--help`, and run the command in a temporary `HOME`, or in `+home` with `fakey`.
  - Screenshots and GIFs come from `demos/` tapes or the TUI screenshot tooling, never
    by hand.
  - When code and docs disagree, record a `PROPOSED FOLLOW-UP:` (bug) note on your phase
    bead. Never add a sentence to the docs explaining the disagreement.
- **Writing style:**
  - Use present tense and plain words.
  - Teach the seven core nouns before using any other term: project → workspace → agent
    → prompt/macro → Patch → bead/plan → TUI.
  - Any other term appears only on the page that needs it, linked to the glossary.
- **Verify** with `sase tool run check`. Once `links` has landed, `just docs-check` also
  runs the strict build and URL contract. Never run `just check-full` unless explicitly
  told to.
- **Phase workers file nothing.** Record discovered work as `PROPOSED FOLLOW-UP:` notes
  on your own phase bead.

## Phase nav-tabs

Restructure the shell without moving a file. Today's pages stay where they are; only
`mkdocs.yml` and the homepage frontmatter change. The homepage body itself is rebuilt
later by `homepage`.

- **Theme features to add** (Material 9.7): `navigation.tabs`, `navigation.tabs.sticky`,
  `navigation.path` (breadcrumbs), `navigation.footer` (previous/next), `toc.follow`,
  and `content.code.copy`. Keep `navigation.indexes`, `navigation.sections` (within
  tabs), `navigation.top`, `search.suggest`, and `search.highlight`.
- **Regroup the nav into Home + six tabs.** Use these interim groupings:
  - **Get started:** Your first 15 minutes (`getting_started.md`), Initialization
    (`init.md`), Shell completion (`completion.md`).
  - **Guides → Fix problems:**
    - `agent_auto_restart.md`, which is linked from the homepage but missing from the
      nav today;
    - `troubleshooting/runner-slots.md`;
    - `troubleshooting/agent-revival.md`.
  - **Concepts:** `architecture.md` (as "How SASE works"), `sdd.md`.
  - **Reference:** every other user-facing page, grouped by product unit:
    - TUI;
    - CLI;
    - Configuration;
    - Macros and workflows;
    - Work units;
    - Agents and workspaces;
    - Automation;
    - Providers;
    - Memory;
    - Prompts and artifacts;
    - Integrations;
    - Migration (`content_layout.md`).
  - **Contribute:**
    - `development.md`, `rust_backend.md`, and `fakey.md`;
    - `telemetry.md`, `perf_runbook.md`, and `mobile_mvp_runbook.md`;
    - `integrations.md`, `artifacts_pane_contract.md`, and
      `artifacts_pane_visual_grammar.md`;
    - `sdd_storage.md`, `instruction_inventory.md`, and `acknowledgements.md`.
  - **Blog.**

  Aim for at most 30 visible items per sidebar. Nest the Reference product groups if
  needed.

- **Homepage frontmatter:** add `hide: [navigation, toc]` to `index.md`.
- **Analytics (R9):** add Material's `extra.analytics` custom-provider hook for
  cookie-less Cloudflare Web Analytics.
  - The partial renders nothing while the token is empty.
  - `mkdocs-pdf.yml` disables the hook.
  - The phase cannot obtain the token. State in the landing summary that Bryan either
    enables Cloudflare's dashboard automatic setup for sase.sh or pastes the beacon
    token into `extra`.
- **Check the mobile drawer separately.** Take a Playwright screenshot at about 390 px
  width of the homepage and one Reference page. Confirm the tabs collapse into the
  drawer and stay usable. Note the result on the phase bead.

**Exit:**

- the strict build passes;
- every existing page is reachable from exactly one tab;
- no sidebar shows more than about 30 items;
- every existing URL is unchanged.

## Phase links

Turn links into a validated interface, and run the docs build before landing as well as
after it.

- **`mkdocs.yml` validation and exclusions:**
  - Add a `validation:` block: `omitted_files`, `absolute_links`, `unrecognized_links`,
    and `anchors` set to `warn`, which `strict: true` turns into failures.
  - Add `exclude_docs` entries for the image briefs (`images/*.prompt.md`,
    `images/*.critique.md`, `images/infographic-style-brief.md`), `_templates/`,
    `_meta/`, `_includes/`, and `README.md`.
  - Use `not_in_nav` only for intentionally unlisted pages.
  - Fix every warning this surfaces, for example `/downloads/...` absolute links in
    `index.md`.
- **`tools/check_docs_urls` (new)** runs against a built site directory. It collects
  URLs from:
  - every `https://sase.sh/...` URL in `src/`, `README.md`, and the README at the latest
    release tag (`git show v<manifest version>:README.md` when the tag is available);
  - `docs/_redirects`;
  - the frozen list `docs/_meta/url_contract.txt`. Seed it now with every currently
    published page URL, every blog post URL, and every README deep link with its anchor.

  For each URL it follows `_redirects` rules, asserts the target exists, and asserts the
  anchor's `id` exists in the target HTML. It leaves an extension point so `moves` can
  add anchor stubs. Tests cover pass and fail cases on a tiny fixture site.

- **Fix the two 404s (bead `sase-1ld`):**
  - In `src/sase/ace/tui/widgets/agent_onboarding.py`, `/macro/` → `/macros/`.
  - In `patch_onboarding.py`, `/patch/` → `/change_spec/`.
  - Add `_redirects` lines for `/macro/` and `/patch/`, because released builds may
    already ship them.
- **Gates:**
  - `just docs-check` runs the strict build, then `tools/check_docs_urls` against the
    built site.
  - Add a light `docs` job to `.github/workflows/master-gate.yml` that runs
    `just docs-check`. It needs no Rust core and no core-wheel dependency.
  - Add a `docs (build + links)` stage to `just check` (via `tools/run_silent`). It runs
    when the change touches `docs/`, `mkdocs*.yml`, `README.md`, or a `src/` file
    containing a sase.sh URL; diff-scoped against the merge base is fine.
  - Full CI's `docs-build` already runs `docs-check` and needs no change.

**Exit:**

- the strict build validates anchors and omitted files;
- the URL contract is green, and the two onboarding links resolve;
- a deliberately broken anchor fails `just check` locally.

## Phase applicability

Make the docs say which product they describe (R6), and make sure the documented first
command installs it (R0). Publishing 0.18.0 itself belongs to `sase-1c1.14`, not to this
epic.

- **Single-source the documented install command.**

  > [!decision] install_command = floor The command is `uv tool install "sase>=0.18"`.
  > While 0.18.0 is unpublished or unresolvable, it fails loudly with a resolver error
  > instead of silently installing 0.1.0. Raise the floor only when the tutorials start
  > depending on a newer release.

  > [!decision] install_command = bare The command is `uv tool install sase`. The
  > scheduled smoke below is the only guard.

  Apply it to `getting_started.md` step 1 and the README install section now. Keep the
  command as one constant in a small tool module that the smoke reads. `start` later
  turns that constant into a `tools/render_docs` source, so `start/install.md` and the
  homepage carry the command in generated blocks.

- **Latest-release line:** add a small, dismissible header notice through the `announce`
  block in `docs/overrides/main.html`, with `announce.dismiss` turned on. Example:
  "These docs track the development branch · Latest release: vX.Y · Install".
  - The version comes from `.release-please-manifest.json` through a generated partial,
    drift-checked by `just fmt-docs-check`.
  - It must stay truthful: if the manifest version is below the install floor, say that
    the next release is pending, not that vX.Y is installable.
  - `mkdocs-pdf.yml` hides it.
- **Scheduled install smoke:** add `.github/workflows/install-smoke.yml` (daily cron
  plus `workflow_dispatch`). On a clean `ubuntu-latest` runner it:
  - runs the documented install command with `uv`;
  - asserts that `sase version -j` reports at least the documented floor (or the
    manifest version under `bare`);
  - runs `sase --help` and `sase doctor -j`, tolerating warnings and failing on hard
    errors.

  It reuses `smoke/pypi/smoke_check.sh` logic where practical but needs no Docker. A red
  run is the signal that the installable product and the docs disagree.

**Exit:**

- one documented install command exists, sourced from one place;
- the version line renders;
- the smoke workflow runs green or reports the real resolver failure;
- the landing summary states the current PyPI resolution result.

## Phase pdf-handbook

The full-corpus handbook is 1,151 pages and 13.9 MiB. It already breached the 22 MiB
`tools/validate_docs_pdf` gate once, takes about 8 of every deploy's 9 minutes, and is
the one part of the stack Zensical cannot build.

> [!decision] pdf_scope = curate The PDF keeps `/downloads/sase-handbook.pdf` and
> contains:
>
> - the homepage;
> - everything under `start/`, `concepts/`, and `guides/`;
> - the launch essay (`blog/posts/structured-agentic-software-engineering.md`);
> - `getting_started.md`, as an interim page while it still exists.
>
> Select pages **by directory rule**, not a hand list, so the Guide phases add pages to
> the PDF with no PDF edits. First find out how `mkdocs-exporter` selects or aggregates
> pages (per-page metadata, aggregator filters, or an `exclude_docs`/nav override in
> `mkdocs-pdf.yml`). Choose the least invasive mechanism that keeps the PDF build
> strict. Links from handbook pages to web-only pages become absolute
> `https://sase.sh/...` links in the PDF (extend `tools/postprocess_docs_pdf` if
> needed).
>
> - Update the front cover template text and the homepage and README handbook
>   descriptions ("Guide-tier handbook", not "all docs").
> - Keep `validate_docs_pdf` and the deploy smoke.
> - Record the new PDF step time and size on the phase bead.

> [!decision] pdf_scope = retire Remove the following:
>
> - the PDF step from `docs-deploy.yml`;
> - `just docs-pdf-check` and the `docs-pdf` dependency group;
> - `mkdocs-pdf.yml`, `docs/templates/pdf/`, `docs/stylesheets/pdf.css`, and
>   `docs/javascripts/pdf-numbering.js`;
> - `tools/postprocess_docs_pdf`, `tools/validate_docs_pdf`, and their tests;
> - the PDF line in `docs-deploy-artifact-check`.
>
> Then:
>
> - add a `_redirects` 302 from `/downloads/sase-handbook.pdf` to `/`;
> - remove the homepage, README, and Full CI references.

> [!decision] pdf_scope = full Keep the full PDF unchanged. Record the decision and the
> current size and step time on the phase bead, and make no build changes.

**Exit:** the deploy builds what the decision says. Under `curate`, the deploy's PDF
step takes under 2 minutes and the PDF is well under the gate.

## Phase contract

Build the "more structure" half of the request.

- **`tools/check_docs`** (stdlib + PyYAML; runs in the format venv) with tests. Wire it
  as `just docs-lint`, and add it to `just check`, `just check-full`, and `just lint` so
  Master Gate runs it. Rules:
  1. **Typing:** a page outside `index.md`, the six typed directories, generated stubs,
     and the legacy allowlist fails. New root pages fail.
  2. **Frontmatter:** typed pages need `title` and a one-line `description` (≤ 160
     chars). Unknown keys fail. `covers:` is allowed only under `reference/`.
  3. **Budgets** per the contract table. Legacy ceilings live in
     `docs/_meta/budgets.yml`. `just fix` lowers a ceiling when a page shrinks and never
     raises one. Generated whole-file pages and `BEGIN GENERATED` blocks don't count.
  4. **Guide hygiene** patterns per the contract summary.
  5. **No test pins Guide prose** (see the contract summary).
  6. **Portability:** no `hooks:` key and no plugin outside the allowlist in
     `mkdocs*.yml`.

  Failure messages say what to do, for example: "Over budget: this page may not grow.
  Put new reference material in a new `reference/` page, or regenerate."

- **Templates** in `docs/_templates/` (tutorial, how-to, concept, reference, hub). Each
  shows the opening sentence, prerequisites, structure, and closing links from the
  contract summary. Tutorial and concept templates include `search: {boost: 2}`.
- **`docs/contribute/index.md`** (a hub linking today's contributor pages in place) and
  **`docs/contribute/writing-docs.md`**, the contract for humans and agents:
  - the six tabs and the directory-is-the-type rule;
  - the Diátaxis compass as a two-question placement test (action or cognition? study or
    work?);
  - the templates, budgets with the ratchet, and hygiene rules;
  - the routing table (see `routing`);
  - the URL compatibility policy;
  - "IA is a human decision: never move pages between tabs without an approved plan."

  Add both to the Contribute tab, and link `development.md`'s docs section to the new
  page instead of restating it.

**Exit:**

- `just check` fails on each rule with a fixture violation;
- the current tree passes with today's untyped pages (the root pages and
  `troubleshooting/*`) allowlisted at their current ceilings.

## Phase routing

Install the accretion guard (R4). It is the one agentic piece that cannot wait, because
feature agents keep writing docs during the whole migration.

> [!decision] docs_routing_note Through `/sase_memory_write`, add
> `sase/memory/docs_writing.md` (`type: reference`). Its description is the trigger:
> "Read before adding, moving, or editing anything under `docs/`, `mkdocs*.yml`, or a
> sase.sh link in code." The body is short:
>
> - **The routing table:**
>   - **A CLI flag, config key, keybinding, directive, or model:** regenerate with
>     `just fix`. Write no prose unless the generated description is insufficient (then
>     improve the source description).
>   - **The behavior of an existing feature:** update its Reference page. If a tutorial
>     or guide step is now wrong, fix that step and nothing more.
>   - **A new user-visible capability:** add a Reference entry (a new `reference/` page
>     if the nearest one is at its ceiling). If it serves a new user goal, note a "guide
>     candidate" `PROPOSED FOLLOW-UP:` or file a task. Never append narrative to a Guide
>     page.
>   - **A design decision:** write a decision record; it appears on the generated
>     design-decisions page.
>   - **Internals, contracts, or runbooks:** `contribute/`.
> - **Budget rule:** legacy pages may not grow.
> - **Pointers:** to `docs/contribute/writing-docs.md` and the docs map.
>
> Then run `sase memory init`. The rule is phrased in tiers and directories so it stays
> true while legacy pages are still at the root.

- **`refresh_docs` chop** (`src/sase/scripts/sase_chop_refresh_docs.py`): keep the
  prompts generic, since they serve every project, and add constraints:
  - If the repository defines a documentation contract (a docs map or a "writing docs"
    page), follow its page types, budgets, and routing.
  - Keep each page in its single mode.
  - Propose a split instead of growing an over-budget page.
  - In tutorial, guide, and concept pages, fix broken steps only.
  - Never move pages between sections or tabs.
  - Report code/docs disagreements instead of documenting them.

  Update the chop's tests.

**Exit:** the note appears in the generated reference-memory list, and the chop's tests
cover the new prompt text.

## Phase moves

Build the machinery every move after this one uses. Do it once, with tests, so the
decomposition is mechanical.

- **`tools/render_docs`:** one renderer framework for every generated docs artifact.
  - **Two kinds of output:** `BEGIN GENERATED` / `END GENERATED` blocks inside
    hand-written pages, and whole generated files carrying a "generated, do not edit"
    marker that `check_docs` recognizes.
  - **Registered sources** declare whether they need `sase` importable:
    - YAML/JSON-only sources run in the format venv through `just fmt-docs` and
      `just fmt-docs-check`;
    - sase-importing sources run in `.venv` through new `just docs-gen` and
      `just docs-gen-check` recipes. Add those to `just fix` and `just check`.
  - Wrap `tools/render_model_docs` as the first registered source. Keep its YAML-only
    guarantee and its tests.
- **`docs/_meta/moves.yml` plus a moves source.** From the map, the source generates:
  - the `_redirects` block, kept between markers so hand lines like `/xprompt/` stay;
  - stub pages for split or decision-cited pages (see the URL policy);
  - the `not_in_nav` block for stubs in `mkdocs.yml`, between markers.

  `check_docs_urls` reads the map to validate anchors through stubs.

- **`tools/docs_split`:**
  - **Inputs:** a source page and a YAML mapping of headings (H2/H3) to target files.
  - **Section moves:** moves each section verbatim and rewrites relative links inside
    the moved text.
  - **Inbound links:** rewrites links elsewhere in `docs/` that pointed at moved
    anchors.
  - **Bookkeeping:** appends the anchor moves to `moves.yml` and prints a summary.
  - Tested on fixtures.
- **Docs map:** a generated `docs/README.md`, excluded from the site and rendered by
  GitHub when browsing `docs/`. It lists every page by tab with its `description` (title
  plus first sentence for legacy pages), and ends with a "Moved pages" table (old path →
  new path) from `moves.yml`. Agents holding stale `docs/…` pointers, for example in
  immutable decision records, find the new home there.
- **`/llms.txt`:** a generated `docs/llms.txt` in llmstxt.org format, with absolute
  sase.sh URLs and descriptions. No plugin.
- **Pilot:** take `instruction_inventory.md` off the site. It is a historical epic
  artifact.
  - Move it to `sdd/history/instruction_inventory.md`.
  - Add a 301 from `/instruction_inventory/` to `/instruction_bundles/`.
  - Repoint the two inbound links in `agent_providers.md` and `init.md`, and remove its
    nav entry.

**Exit:**

- `just fix` regenerates everything, and `just check` reports drift;
- the pilot move passes the URL contract;
- a `docs_split` fixture round-trip keeps every anchor resolvable.

## Phase glossary

- **Glossary source:**
  - It reads `sase/memory/glossary/*.md` strands directly. That is tooling, not an agent
    read.
  - It renders `docs/reference/glossary.md`: every term, its aliases, and its
    definition, with anchors named after term slugs.
  - Agent-internal cross-reference syntax becomes plain links.
- **Design-decisions source:** reads `sase/memory/decisions/*.md`. It renders
  `docs/reference/design-decisions.md`, a roster with each record's claim, status
  (including superseded marks), and a link to the record on GitHub. Concept pages link
  to it for "why".
- **Tooltips:**
  - Generate `docs/_includes/abbreviations.md` for about 15 core terms: the seven nouns,
    plus Patch, bead, plan, epic, macro, gate, monitor, and stitch. Keep the term list
    in the renderer config.
  - Each entry's text is the term's one-line definition.
  - Enable the `abbr` and `pymdownx.snippets` extensions with `auto_append` in
    `mkdocs.yml`. Hovering "Patch" in Guide text then shows the definition.
- **`docs/reference/index.md`:** the Reference routing hub, at most 400 words. It is
  organized by product unit (CLI, Configuration, TUI, Macros and workflows, Work units,
  Providers, Memory, Automation, File formats, Glossary, Design decisions) and links to
  today's pages in place. Later phases repoint its links as pages move.

**Exit:**

- both pages are generated and drift-checked;
- tooltips render on a Guide fixture page;
- the Reference tab opens on the hub.

## Phase start

Write the Get started tier as new files. The source is `getting_started.md`, which grew
from 1.5K to 4.1K words by absorbing provider-detection exceptions and size-alias
routing. Those details belong in guides and the Reference.

- **`start/index.md`** (hub): "Is SASE for you?":
  - one paragraph on what SASE does;
  - alpha status;
  - requirements (OS, Python, `uv`, one authenticated agent CLI);
  - the two lessons and the time each takes.
- **`start/install.md`:**
  - the documented install command, then `sase version`. The command sits in a
    `BEGIN GENERATED: install-command` block rendered by a new `tools/render_docs`
    source that reads the constant from `applicability`;
  - one authenticated provider;
  - `sase doctor`, with recovery links into the Reference;
  - shell completion as an optional last step.
- **`start/first-run.md`** (tutorial 1; target 900–1,500 words, cap 2,500):
  1. Confirm the install.
  2. Launch a read-only run in the managed `+home` project. Say plainly that `+home` is
     SASE's demo workspace, not the current directory.
  3. Open `sase tui`, find the run, and identify its prompt, status, and result.

  Finish with "You have launched and inspected a SASE run," then two next choices. Out
  of scope: macro syntax, the glossary, artifact digest forms, dependency scheduling,
  autonomy, and provider-selection internals. Link those at the point of need instead.

- **`start/first-change.md`** (tutorial 2):
  1. Make a small edit in the demo project.
  2. Inspect the diff.
  3. Explain what landing will commit or push **before** the step that triggers it.

  Describe workspace isolation accurately: telling an agent not to edit files is not an
  enforced guarantee.

- **Marking tutorial blocks.** Every shell block in `start/` is a tutorial step by
  default. Mark exceptions with `<!-- tutorial-test: skip <reason> -->` and expected
  output with `<!-- tutorial-test: expect "<substring>" -->` on the line before a block.
  `tutorial-tests` executes these markers.
- **Moves:**
  - `getting_started.md` → `start/` through `moves.yml`: a 301 to `/start/`, or a stub
    if the URL contract finds its anchors deep-linked.
  - Delete the draft duplicate `blog/posts/hello-sase-your-first-15-minutes.md` and
    repoint its `_redirects` line to `/start/`.
  - Convert `tests/test_docs_getting_started_providers.py` so it no longer pins Guide
    prose: drop it, or point it at a generated or Reference source.
  - Point the install smoke and the PDF interim rule at the new pages.
- **Nav:** the Get started tab becomes exactly the `start/` pages. The interim entries
  `init.md` and `completion.md` move to the Reference tab. The PDF selection picks up
  `start/` automatically under `curate`. The homepage's "Getting Started" button keeps
  working through the redirect.

**Exit:**

- `start/` totals ≤ 10K words and every page is within budget;
- following the pages by hand in a temporary `HOME` works;
- README, onboarding, and URL-contract links to `/getting_started/` still resolve.

## Phase tutorial-tests

Tutorials must aim for "perfect reliability", so they are tested.

1. **Feasibility.** Start with a short spike recorded on the phase bead: what `fakey`,
   the HOME-isolation fixtures, and `+home` allow, and which steps (TUI inspection) need
   a CLI-equivalent assertion instead.
2. **Runner.** Build `tests/docs/test_tutorials.py`, or a `tools/run_tutorials` driver
   with a thin pytest wrapper. It:
   - parses `docs/start/*.md` in order and executes unskipped shell blocks in a fresh
     temporary `HOME` and scratch directory;
   - routes agent launches to `fakey-large` through configuration or environment, never
     by editing the page text;
   - asserts exit codes and `expect` markers;
   - reports the failing page, step number, and block text.
3. **CI.** Mark it so the fast lanes skip it, and add it to the scheduled Full CI lane
   in `ci.yml`. If the screenshot pipeline makes it cheap, add a `demos/` VHS tape for
   tutorial 1's TUI step.
4. **Doc note.** Add a short "Testing the tutorials" section to `writing-docs.md`:
   - an occasional real-provider walkthrough still covers authentication and routing,
     which `fakey` cannot;
   - list the walkthrough as a manual checklist item for Bryan.

**Exit:**

- both tutorials pass in the runner;
- a deliberately broken step fails with a readable message;
- the Full CI lane includes the runner.

## Phase concepts-model

Concepts teach a small model first. The glossary has about 70 terms; the Guide teaches
seven nouns before it uses any other.

- **`concepts/index.md`, "How SASE works"** (about two screens):
  - **One diagram.** Prefer a Mermaid diagram through a `pymdownx.superfences` custom
    fence, which is a Markdown extension, not a plugin.
  - **The seven nouns,** each in one or two sentences, in order: project → workspace →
    agent → prompt/macro → Patch → bead/plan → TUI.
  - **A "Where next" block.**

  Every "why?" link on the site and the homepage's secondary CTA point here.

- **`concepts/agents.md`:** agents, sessions, and workspaces.
- **`concepts/work-units.md`:** Patches, beads, plans, and goals, and when to use each.
- **`concepts/landing.md`:** how changes land: commits, stitches, finalizers, and VCS
  providers, at the concept level.
- **Every concept page** follows the concept template and ends with "Do it" (link guides
  as they exist; later guide phases add theirs) and "Look it up" (Reference) links.
- **Sources:** extract the explanation from `architecture.md` and the component map that
  `start` left behind, and link to `agent_sessions.md`, `workspace.md`, `sdd.md`,
  `change_spec.md`, `beads.md`, `goals.md`, `commit_workflows.md`, and `vcs.md`.
- **Move `architecture.md`:** its internals (source layout, daemon and provider
  boundaries) move to `contribute/architecture.md`, and the old URL redirects through
  `moves.yml`. Repoint homepage links.
- **Nav:** the Concepts tab becomes exactly the `concepts/` pages. Move the interim
  `sdd.md` entry to the Reference tab.

**Exit:**

- four pages, each within budget and hygiene-clean;
- `architecture.md` moved with the URL contract green.

## Phase concepts-system

Use the same template and rules as `concepts-model`.

- **`concepts/prompts.md`:** prompts, macros, and workflows.
- **`concepts/memory.md`:** memory and agent instructions, at the concept level. Bundle
  delivery internals stay Reference or Contribute.
- **`concepts/automation.md`:** automation and human checkpoints: the scheduler,
  mentors, gates, plan approvals, and autonomy profiles. Explain why checkpoints exist.
- **`concepts/providers.md`:** providers and models, including size aliases at the
  concept level only. The generated model tables stay in the Reference.

**Exit:** the Concepts tier is complete (8 pages, ≤ 25K words total), and each page
links to the glossary and the design-decisions page where relevant.

## Phase guides-setup

- **`guides/index.md`:** a goal list grouped as Set up, Daily work, Plan and automate,
  and Fix problems. List only guides that exist; later phases append theirs. It is "an
  encouraging suggestion of capabilities", not a complete map.
- **Set up:**
  - **Add an existing repository** (from `init.md`; migration notes go toward
    `reference/upgrading.md`).
  - **Choose providers and models** (from `agent_providers.md` and `llms.md`'s user
    sections).
  - **Configure the essentials** (about 1.5K words; links to the generated config
    reference once it exists, legacy `configuration.md` anchors until then).
  - **Set up shell completion and your editor** (from `completion.md` and `editor.md`).
- **Fix problems:**
  - **Start with sase doctor.**
  - **Unstick a queued agent** (from `troubleshooting/runner-slots.md`).
  - **Recover an interrupted agent** (from `agent_auto_restart.md` and
    `troubleshooting/agent-revival.md`).
- **Move the source pages.** `troubleshooting/*` and `agent_auto_restart.md` move out of
  the root:
  - how-to narrative → the guides;
  - remaining facts → `reference/` pages, for example `reference/agent-auto-restart.md`.

  Record both in `moves.yml`; the README links `/agent_auto_restart/`.

- **Update links elsewhere:**
  - point `start/` "Next" links at "Add an existing repository";
  - add "Do it" links to the concept pages.

**Exit:**

- the guides exist, each within budget with a verb-led goal title;
- the moved pages pass the URL contract;
- the interim Fix problems nav entries now point at the new guides.

## Phase guides-daily

- **Run agents in parallel:** fan-out, isolated workspaces, and watching the runs.
- **Review and land a change.**
- **Hand a result to another agent:** artifact references, at task level.
- **Reuse a prompt with macros.**
- **Find and replay an old prompt:** from `prompt.md` and the Prompts overlay in
  `ace.md`.

Sources: `ace.md`, `macros.md`, `prompt.md`, `change_spec.md`, `commit_workflows.md`,
`artifact_references.md`. Extract how-to narrative only. Tables, keymaps, and syntax
stay in the Reference and are linked. Append to the guides hub and add "Do it" links to
the matching concept pages.

**Exit:** five guides within budget; each legacy passage they replace is now a link.

## Phase guides-automate

- **Plan a multi-agent change:** plans, epics, and plan decisions, from `sdd.md` and
  `beads.md`.
- **Schedule recurring work:** from `axe.md` and `workflow_spec.md`.
- **Get automated code review:** from `mentors.md`.

Same rules as `guides-daily`. A third tutorial (plan → epic) is explicitly deferred
until readers have been observed on the first two.

**Exit:** the Guide tier is complete at about 30K words, with every page within budget.

## Phase cli-registry

- **Generated pages:** generate `docs/reference/cli/<group>.md`, one page per top-level
  command group, plus `docs/reference/cli/index.md`. The source is the argparse tree;
  use `sase.completion.snapshot` or the parser directly. Each page lists:
  - usage and description;
  - subcommands;
  - an options table with long and short flags, defaults, and help text;
  - the bare-`list` default where it applies.

  It is a `.venv` source (`just docs-gen`), and `just docs-gen-check` catches drift.

- **Trim `cli.md`:** delete its hand-written tables and stale caveats that the generated
  pages now cover. `cli.md` stays the routing index at its current path (`decompose`
  moves it), with links to the generated pages.
- **Help-text quality:** where help text is too thin to stand alone, improve the
  argparse help (per `cli_rules.md`), not the docs.

**Exit:** no hand-maintained CLI flag tables remain, and adding a flag without
regenerating fails `just check`.

## Phase config-registry

- **Generated pages:** generate `docs/reference/config/<family>.md` plus an index from
  `src/sase/config/sase.schema.json`: 65 top-level keys, 1,239 nodes, 71% described.
  Defaults come from `src/sase/default_config.yml`. Group small top-level keys into a
  few family pages and give large families (`ace`, `axe`, `llm_provider`, `memory`, …)
  their own. Each entry shows:
  - the key path and type;
  - the default and allowed values;
  - the description.

  This is a YAML/JSON-only source.

- **Trim `configuration.md`:** delete the text that only restates schema facts, and
  replace each deleted region with a link to the generated entry. Prose the schema
  cannot say stays for `decompose`.
- **Missing descriptions:** list keys without a schema description in a generated "needs
  description" report. Fill the most-used ones in the schema; leave the rest as a
  follow-up note.
- **Configure the essentials:** repoint that guide's links to the generated pages.

**Exit:** `configuration.md` shrinks substantially, its ceiling ratchets down, and
schema drift fails `just check`.

## Phase keymap-directives

- **Keymap:** generate `docs/reference/tui/keymap.md` from `default_config.yml`
  `ace.keymaps` (about 505 bindings), one table per mode or pane. Delete the roughly 475
  hand-written keymap rows in `ace.md` and link the generated page instead. This is a
  YAML-only source. Per `gotchas.md`, keymap changes already go through
  `default_config.yml`.
- **Directives:** generate `docs/reference/macros/directives.md` from the Rust directive
  contract (`require_rust_binding("directive_contract")`, a `.venv` source):
  - name, arguments, and summary;
  - visibility, including feature-flag gating.
- **Discovery order:** generate the macro discovery-order table from the macro-source
  resolver. Replace the drifted table in `macros.md`, which is bead `sase-1la`.

**Exit:** no hand-maintained keymap or directive tables remain, and changing a binding
or directive without regenerating fails `just check`.

## Phase homepage

The homepage is one screen with one primary action.

- **Hero.** This copy is a draft; Bryan may edit it after landing:
  - **Headline:** "Run a team of coding agents. Keep every change reviewable."
  - **Subhead:** "SASE runs Claude Code, Codex, and other agent CLIs in parallel,
    isolated workspaces, and lets you track, review, and land their work from one
    terminal UI."
  - **Status pill:** "Alpha · Linux & macOS · needs one authenticated agent CLI". Verify
    the OS claim.
  - **Install block:** three lines with copy buttons: the documented install command,
    `sase doctor`, `sase tui`.
  - **Primary CTA:** "Run your first agent (15 min)" → `start/first-run/`.
  - **Secondary CTA:** "How SASE works" → `concepts/`. GitHub stays the header icon.
- **One demo.** Use `demos/out/sase_ace_multi_model_fanout.mp4`, not the 12 MB GIF:
  - a poster still, alt text, and controls;
  - no forced autoplay.
- **Four capability cards,** each linking a Concept page, never a giant Reference page:
  - parallel isolated runs;
  - supervise from one TUI;
  - durable, reviewable work;
  - reusable prompts and automation.
- **"How these docs are organized":** four doors, one line each. Learn → Get started, Do
  → Guides, Understand → Concepts, Look up → Reference.
- **Footer row:** launch essay · blog · handbook PDF (per `pdf_scope`) ·
  acknowledgements.
- **Cleanup:** remove the role cards, primitives list, and next-click grid, and trim the
  `sase-*` CSS in `extra.css` to what remains. Keep the social meta in
  `overrides/main.html` and the `docs-deploy-artifact-check` assertions intact.
- **Front-door links:**
  - Rewrite the README "Documentation" section around the four doors and a few key
    guides. README is also the PyPI long description, so every old URL must keep
    resolving through the contract.
  - Retarget the TUI onboarding widgets (`agent_onboarding.py`, `axe_onboarding.py`,
    `patch_onboarding.py`) and `parser_sudo.py` to the most specific Guide or Concept
    page. Update their tests.
- **Reader-testing kit (R9):** add `docs/contribute/reader-testing.md`, a 3–5 person
  observed round:
  - six named tasks, for example: "Is SASE for me?", run a first agent, run two agents
    in parallel, "what is a bead?", find a config key, recover an interrupted agent;
  - the success measure: at least 4 of 5 readers pick the right door first;
  - a results template, and how label changes are proposed afterwards.

  Bryan runs the round. Its results become follow-up work.

**Exit:**

- one primary CTA, one secondary CTA, and four doors;
- install and demo above the fold at 1280×800;
- mobile checked;
- a strict build with the URL contract green.

## Phase decompose

> [!decision] docs_pointer_notes

This is an `xlarge` phase: its worker authors an epic plan and runs it as a sub-epic. It
covers the research report's phase 3, splitting the Reference bottom-up, one page group
per sub-phase, largest and most-read first.

**Sub-phase rules:**

- **Move first, edit second.** A move-only commit uses `tools/docs_split` and
  `moves.yml`, and is followed by content commits. Land each sub-phase quickly.
- **Split boundaries.** Split by complete task, coherent concept, or one reference
  family, never by individual flag. Keep a working page intact if splitting would make
  its task harder to follow.
- **History** ("formerly", "no longer", rename notes, epic labels) moves to
  `reference/upgrading.md` or is dropped where the CHANGELOG already records it.
- **Ceilings.** Every split page ends up within budget, and its legacy ceiling entry is
  removed.

**Starting dispositions** (G = already handled by a Guide phase; R = Reference; C =
Contribute):

| Current page(s)                                                                                                                | Destination                                                                                                    |
| ------------------------------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------- |
| `ace.md` (103K)                                                                                                                | R: `reference/tui/`, one page per TUI tab plus the generated keymap; visual-grammar remnants → C               |
| `configuration.md` (71K)                                                                                                       | R: prose into `reference/config/` family pages; `reference/environment-variables.md`                           |
| `macros.md` (32K), `workflow_spec.md`                                                                                          | R: `reference/macros/` (syntax, discovery, generated directives), `reference/workflows.md`                     |
| `beads.md` (28K), `sdd.md`, `goals.md`, `change_spec.md`, `project_spec.md`                                                    | R: `reference/work-units/` (lifecycles, formats, CLI pointers)                                                 |
| `llms.md` (27K), `agent_providers.md`, `instruction_bundles.md`, `memory.md`, `memory_history.md`                              | R: `reference/providers/`, `reference/memory/`; instruction-delivery and manifest internals → C                |
| `notifications.md` (18.5K), `axe.md` (17K), `monitors.md`, `tool.md`, `mentors.md`, `sudo.md`                                  | R: `reference/automation/`; epic-labelled sections → C                                                         |
| `cli.md`                                                                                                                       | R: becomes `reference/cli/index.md`; merge its routing role into `reference/index.md`                          |
| `agent_sessions.md`, `workspace.md`, `vcs.md`, `commit_workflows.md`, `agents_sidecar.md`, `init.md`, `content_layout.md`      | R: `reference/agents/` and `reference/vcs/`; provider APIs, orchestrator, sync → C; migration → `upgrading.md` |
| `plugins.md`, `editor.md`, `completion.md`, `mobile_gateway.md`, `remote_dispatch.md`                                          | R: `reference/integrations/`; API details → C                                                                  |
| `query_language.md`, `agent_images.md`, `pager.md`, `prompt.md`, `artifact_references.md`, `artifact_links.md`                 | R: `reference/` (one page each, split where over budget)                                                       |
| `development.md`, `rust_backend.md`, `fakey.md`, `telemetry.md`, `perf_runbook.md`, `mobile_mvp_runbook.md`, `integrations.md` | C: `contribute/` (moves; warn-only budgets)                                                                    |
| `artifacts_pane_contract.md`, `artifacts_pane_visual_grammar.md`, `sdd_storage.md`, `acknowledgements.md`                      | C: `contribute/`                                                                                               |

**Also in scope:**

- `reference/upgrading.md`, the single history home;
- final versions of the `reference/index.md` and `contribute/index.md` hubs;
- repointing README links to canonical URLs;
- updating `src/` path references (about 22 files name `docs/*.md`);
- a last pass of the docs map.
- **Memory pointers** (`docs_pointer_notes`). Through `/sase_memory_write`, update the
  plain-text `docs/…` pointers when their pages move, then run `sase memory init`:
  - `tui_perf.md` → `perf_runbook.md`;
  - `dispatch.md` → `remote_dispatch.md`;
  - `macros.md` → `macros.md` and `cli.md`;
  - `cli_rules.md` → `plugins.md`.

  Decision records are immutable and are not edited. The core note
  `rust_core_backend_boundary.md` is outside this plan's memory decisions. `tool.md`,
  `macros.md`, and `rust_backend.md` therefore keep stubs at their old paths, and the
  core note's pointer gets a `PROPOSED FOLLOW-UP:` note. The docs map's "Moved pages"
  table covers everything else.

**Exit criteria (whole sub-epic):**

- no root pages remain except `index.md` and generated stubs;
- the legacy allowlist in `budgets.yml` is empty;
- no `reference/` page is over budget;
- the URL contract is green;
- the success-metrics table below is measured and recorded on the epic bead.

## Out of scope and deferred follow-ups

Record these at epic landing. They are not phases here:

- **Publishing 0.18.0.** Owned by `sase-1c1.14`; the PyPI bug is `sase-16p`.
- **Zensical trial.** Run it after Zensical 0.1.0 (planned 2026-11-05) and decide by
  about 2027-02, before Material's critical-fix end on 2027-05-05.
  - **Acceptance cases:** blog, RSS, redirects and anchor stubs, the `custom_dir`
    override, search quality, mobile nav, priority journeys, and built-in `llmstxt` with
    "Copy as Markdown".
  - Under `curate`, the PDF then needs a standalone headless-browser print step, or
    retirement.
  - Fall back to Astro Starlight only if the trial finds durable blockers.
- **Building sase.sh from release tags,** with `master` on a preview URL. Revisit once
  external users and regular releases exist. Don't build a multi-version archive.
- **A weekly docs-editor routine** that reads the CHANGELOG, tutorial-test results, and
  guide candidates, and proposes a plan for human approval. No docs knowledge graph,
  retrieval layer, or support bot.
- **A separate reference site,** only if the Reference still dominates search after
  generation.

## Success metrics

| Metric                                                 | Today                                 | Target                                    |
| ------------------------------------------------------ | ------------------------------------- | ----------------------------------------- |
| Documented install installs the documented version     | No (`uv tool install sase` → 0.1.0)   | Yes, smoke-checked daily                  |
| Homepage primary choices                               | 4 buttons + 16 cards                  | 1 primary CTA + 1 secondary + 4 doors     |
| Sidebar items on any page                              | ~60                                   | ≤ 30 in Guides and Reference, ≤ 10 others |
| Guide-tier words                                       | ~4K of real tutorial                  | ~30K at launch, ≤ 60K ceiling             |
| Hand-written monthly growth                            | +37%                                  | ≤ +5%                                     |
| Tutorial reliability                                   | Untested                              | Green in Full CI                          |
| History phrases / bead IDs / `src/` paths in the Guide | 741 / 148 / 382 across the corpus     | 0 in the Guide                            |
| Hand-maintained registry tables                        | CLI, config, keymaps, directives      | 0                                         |
| Broken sase.sh URLs in code and README                 | 2 known                               | 0, enforced                               |
| PDF                                                    | 1,151 pp, 13.9 MiB, ~8 min per deploy | Per `pdf_scope`; well under the gate      |
| Reader evidence                                        | None                                  | Analytics plus one observed round         |

## Risks

- **Tabs feel abstract to readers.** The editorial types stay; the observed round
  decides the door labels.
- **Too many tiny pages.** Merge pages that readers flip between. Split by task,
  concept, or reference family, never by flag.
- **Duplicate truth during migration.** Extract, don't copy: a replaced passage becomes
  a link in the same commit.
- **Collisions with live agents.** New files for the Guide, move-only commits for the
  Reference, small commits on hot pages, and the routing rule landing early.
- **The ratchet frustrates feature agents.** The failure message names the remedy (a new
  Reference page, or regenerate). Watch for agents raising ceilings by hand: only a
  human-reviewed change may raise one.
