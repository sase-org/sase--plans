---
tier: epic
title: Group-aware agent completion for %wait and
goal: 'Completion menus in the ACE prompt input and the xprompt LSP offer agents,
  agent families, agent clans, and @tribes for %wait and #fork, with each entry''s
  kind unmistakable at a glance.

  '
phases:
- id: lsp
  title: Kind-aware agent catalog and LSP completion
  depends_on: []
  description: '''Kind-aware agent catalog and LSP completion'' section: extend the
    editor helper agent catalog with family/clan/tribe entries and teach the Rust
    xprompt LSP to complete and render all four reference kinds for %wait arguments
    and agent-typed xprompt arguments.'
- id: tui
  title: Group-aware prompt-input completion menu
  depends_on: []
  description: '''Group-aware prompt-input completion menu'' section: add family/clan/tribe
    candidates with distinct, aligned completion rows to the ACE prompt input for
    %wait arguments and agent-typed xprompt arguments.'
create_time: 2026-07-19 12:50:46
status: wip
bead_id: sase-7h
---

# Plan: Group-aware agent completion for %wait and #fork

## Context and outcome

Both `%wait` resolution and the `#fork` workflow accept four reference kinds: a real agent name, an agent-family
container name, a bare agent-clan name, and an `@tribe` reference. Completion currently exposes only a fraction of that.
The ACE prompt input completes visible agents plus tribe tags for `%wait` and agent-typed xprompt arguments (`#fork:` /
`#fork(name=...)`), but offers no clans, renders tribes as generic directive-argument rows, and shows family containers
indistinguishably from plain agents. The Rust xprompt LSP (`sase_xprompt_lsp` in the linked sase-core repository)
completes only a flat agent list for agent-typed xprompt arguments via the editor helper bridge's agent catalog, and for
`%wait` arguments offers only static placeholder suggestions ("agent", "time=5m", "time=1h") with no real names at all.

The finished feature presents one coherent completion vocabulary on both surfaces: typing after `%wait:` / `%wait(...)`
or inside an agent-typed xprompt argument offers agents, families, clans, and tribes, each visually labeled with its
kind, member information for groups, and the established accent styling, so choosing between `review`, `review--fix`,
`myclan`, and `@mytribe` never requires guessing what each name is.

## Shared reference vocabulary

Both phases implement exactly this contract so the two surfaces stay interchangeable:

- **Kinds and insertions.** `agent` inserts the agent's prompt-referenceable name; `family` inserts the bare family
  container name; `clan` inserts the bare clan name; `tribe` inserts `@<tribe>`. Wait keyword candidates (`time=`,
  `runners=`) remain available for `%wait` and stay ahead of name candidates.
- **Matching.** Case-insensitive prefix matching. Tribes match both spellings — a bare prefix matches the tribe name and
  a leading `@` filters to tribes — but always insert the canonical `@<tribe>` form. Candidates are de-duplicated by
  insertion text, and values already selected in a multi-value clause (repeatable `#fork` arguments, comma-separated
  `%wait` clauses) are excluded.
- **Ordering.** For `%wait`: matching keywords, then tribes, clans, families, and agents; within a kind, keep the stable
  display order the source already provides. Agent-typed xprompt arguments use the same order minus keywords.
- **Group metadata.** Clans report the member count and aggregate activity of their newest generation, consistent with
  Agents-tab clan presentation and fork-scope membership (synthetic containers are never members). Tribes report how
  many agents and clans currently carry the tribe. Families report their member count. Groups that would resolve to zero
  real members are omitted rather than offered as dead completions.

## Kind-aware agent catalog and LSP completion

Work in this phase spans the Python editor helper bridge in this repository and the Rust core/LSP crates in the linked
sase-core repository, opened through the sase repo workflow.

**Bridge catalog.** Extend `agent_catalog_response` to emit family, clan, and tribe entries alongside agents, derived
from the same single artifact-scan snapshot the listing already acquires. Classification must agree with wait dependency
resolution: clan names and generations from clan metadata, tribes from persisted agent tags, `%tribe` assignments, and
effective clan tribes, families from the family name grammar. Entries gain additive fields (kind, member count, and a
short human detail); the schema version stays 1 and both directions must degrade gracefully — an older LSP ignores the
new fields, and a newer LSP treats entries without a kind as plain agents. Never let group derivation errors prevent the
catalog from returning agents.

**Rust core completion.** Extend the agent wire entry and `build_agent_completion_candidates` in the sase-core editor
module to carry and honor kinds: distinct candidate kind strings, group label details (for example "clan · 3 members"),
and tribe filter text covering both spellings. Add a `%wait` argument builder that merges keyword candidates (`time=`,
`runners=`, replacing today's placeholder trio) with the kind-aware catalog candidates. Bring `%wait` argument context
detection to parity with the prompt input: comma-separated paren and colon clauses complete only the active clause and
exclude already-selected values, mirroring the existing clan clause narrowing.

**LSP server and rendering.** Route `%wait` argument completion through the asynchronous agent-catalog path the
agent-typed xprompt argument context already uses, preserving the existing timeout, warn-once, and empty-response
degradation so a slow or missing helper never blocks the editor. In the LSP conversion layer, map each kind to a
distinct `CompletionItemKind` with label details naming the kind and group metadata, and set sort text so the shared
ordering survives client-side sorting. Update the deep doctor check for the xprompt LSP if it asserts on catalog or
completion shapes.

**Verification.** Rust unit tests cover kind-aware candidate building, wait clause narrowing, selected-value exclusion,
keyword merging, and LSP item conversion; run the touched sase-core crates' test suites there. Python tests cover group
derivation in the bridge catalog (including tolerance for records missing group metadata) and the schema-compatibility
contract.

## Group-aware prompt-input completion menu

Work in this phase is confined to the ACE TUI in this repository.

**Candidate model and derivation.** Extend the shared agent completion candidate model with a kind plus group metadata
(member count, aggregate status) so one model flows through filtering, soft completion, and menu rendering. Derive
family, clan, and tribe candidates in the app-level provider from the same visible-agents snapshot the current builder
uses: clan candidates from clan containers and member rows (newest generation, per the shared vocabulary), tribe
candidates from agent and clan tribe tags, family candidates from family roots now labeled with their kind and member
count. Synthetic clan containers are clan sources but never agent candidates. Keep the keystroke path read-only: no new
disk reads or globs — raw prompt inspection stays behind the existing mtime-keyed cache, and the per-completion-session
candidate snapshot caching is preserved.

**Menu construction.** Rework the wait/fork argument candidate builder to emit all four kinds per the shared vocabulary,
honoring excluded names for groups as well as agents, and keeping `%wait` keyword candidates first. Update
completion-menu border titles and any command/help wording so the menus describe agents, families, clans, and tribes
(for example "wait targets" and "fork targets") per the help-sync conventions.

**Row rendering.** Give every entry an aligned anatomy: a one-character kind glyph, the name in the existing padded name
column, the existing badge column, then dim context. Agent rows keep their current presentation (status dot, VCS
workflow badge, prompt snippet). Family, clan, and tribe rows show a kind-distinct glyph and accent color, a badge
naming the kind with its member count, and a dim member preview as context. Reuse established styling so the menu
matches the rest of the app: tribe references render as `@name` in the existing bold #FFD75F accent, clan rows echo
Agents-tab clan container styling, and aggregate status uses the shared status styles. Selection styling must remain
legible for all kinds.

**Verification.** Unit tests cover kind derivation from visible agents, ordering, both tribe spellings, exclusion and
de-duplication, empty-group omission, and unchanged single-agent behavior; rendering tests cover each kind's row. Add
PNG visual snapshots of the completion menu showing all four kinds together for both a `%wait` and a `#fork` context,
following the visual snapshot workflow. Existing wait, fork, and soft-completion tests must stay green.

## Risks and constraints

- **TUI responsiveness.** Completion runs on the keystroke path; all candidate data must come from already-loaded state
  and existing caches, per the TUI performance rules. No resolver calls, subprocesses, or unbounded locks.
- **Wire compatibility.** Mixed-version helper/LSP pairs must degrade to today's flat agent behavior, never to an error
  or an empty menu caused solely by schema drift.
- **No behavior change outside completion.** `%wait`/`#fork` parsing, resolution, and launch semantics are untouched;
  this feature only changes what the menus offer and how entries look.
