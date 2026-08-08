---
tier: epic
title: Project-local glossary memory and editor semantics
goal: Make one project-local glossary configuration the reliable source for generated
  agent memory, project-aware prompt highlighting, definition previews, and definition
  editing in ACE and every SASE LSP client.
phases:
- id: glossary-core-contract
  title: Define the canonical glossary domain
  depends_on: []
  description: 'core: add validated glossary parsing, effective aliases, deterministic
    matching, source metadata wires, and reusable Rust/Python APIs.'
  size: medium
- id: glossary-config-memory
  title: Generate glossary memory from project config
  depends_on:
  - glossary-core-contract
  description: 'memory: add the project-local schema and make memory init render glossary.md
    before composing agent instructions.'
  size: medium
- id: glossary-project-catalog
  title: Build project-aware glossary catalogs
  depends_on:
  - glossary-core-contract
  - glossary-config-memory
  description: 'catalog: resolve glossary entries and editable source locations for
    the project selected by prompt VCS context without blocking keystrokes.'
  size: medium
- id: glossary-ace-experience
  title: Add beautiful ACE glossary interactions
  depends_on:
  - glossary-project-catalog
  description: 'ace: highlight glossary aliases and route K and Ctrl+] to project
    glossary previews and source editing.'
  size: medium
- id: glossary-lsp-experience
  title: Add glossary semantics to the xprompt LSP
  depends_on:
  - glossary-project-catalog
  description: 'lsp: expose project glossary aliases through semantic tokens, hover,
    and go-to-definition with live cache invalidation.'
  size: medium
- id: glossary-migration-verification
  title: Migrate SASE's glossary and prove the complete feature
  depends_on:
  - glossary-config-memory
  - glossary-ace-experience
  - glossary-lsp-experience
  description: 'migration: move the existing SASE glossary into config, regenerate
    memory and instruction files, document the workflow, and run cross-repository
    verification.'
  size: medium
proposed_by: bbugyi200.athena.w2
create_time: 2026-08-08 17:02:21
status: wip
bead_id: sase-hq
---

- **PROMPT:** [prompts/202608/project_glossary.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/project_glossary.md)
- **BEAD:** [sase-hq](https://github.com/sase-org/sase--beads/blob/main/pages/sase-hq/README.md)

# Project-local glossary memory and editor semantics

## Outcome and design principles

Add a top-level `glossary` field that is valid only in a project's canonical
`sase/sase.yml`. It becomes the single authored source for domain terminology. A
successful `sase memory init` deterministically generates `sase/memory/glossary.md` from
that field early enough that the same invocation uses the new glossary body when it
renders `AGENTS.md`, provider shims, and the memory README.

The same normalized glossary catalog powers both ACE and the Rust xprompt LSP. There
must not be separate Python and Rust interpretations of aliases, phrase boundaries,
precedence, or collisions. A leading VCS workflow reference in the prompt selects the
glossary's project; otherwise the active workspace project is used, matching the
existing xprompt-assist and artifact-reference precedence. Unknown, disabled, home, or
unreadable project contexts degrade to no glossary semantics instead of leaking terms
from another project.

## Configuration contract

Use a mapping keyed by the canonical displayed term, so authors never duplicate the
name:

```yaml
glossary:
  Agent Clan:
    aliases:
      - agent clans
      - clan
    definition: >-
      A named, rootless container for agents that run in parallel.
```

Each value is a closed mapping with a required nonblank Markdown `definition` and an
optional `aliases` list, defaulting to empty. Each configured alias is a single
nonblank, single-line string containing one or more words; surrounding and repeated
horizontal whitespace is normalized. Retrieval returns the canonical term itself first
in the effective alias list, followed by configured aliases in authored order with
normalized duplicates removed. Matching is Unicode-aware and case-insensitive, requires
word boundaries at word-like ends, allows horizontal whitespace runs inside multiword
phrases, and never crosses a line.

Reject ambiguous catalogs before they can generate memory or drive navigation: blank
terms/definitions/aliases, multiline aliases, duplicate terms after normalization, and
any effective alias claimed by more than one term are errors with a precise
`glossary.<term>...` path. Overlapping but nonidentical aliases are legal; the longest
match wins and ties follow authored order. Matching skips fenced and inline code literal
zones and returns nonoverlapping UTF-8-safe spans with the canonical entry identity. The
JSON schema, Rust validation, Config Center field model, examples in
`default_config.yml`, and focused schema tests must agree. Non-project layers that
contain `glossary` receive an actionable scope diagnostic rather than silently becoming
global glossary sources.

## Generated-memory lifecycle

Treat `sase/memory/glossary.md` as an overwrite-managed short memory note only when a
nonempty project glossary exists. Render stable frontmatter identifying it as SASE
glossary output, a concise glossary title, and one entry per authored term. Preserve
definition Markdown, show only additional configured aliases in an unobtrusive
`Aliases:` line, escape generated labels safely, and pass the result through the
existing Markdown formatter so it remains a Prettier fixpoint. Empty or absent glossary
configuration produces no empty note and retires only a file carrying the
generated-glossary ownership marker. Never delete an unmarked human note; a configured
glossary colliding with an unmarked `glossary.md` must block with migration guidance
rather than overwrite user content.

Load and validate the project glossary with the other memory-init inputs. Render its
expected content before `_amd_sync_plan`, add its fresh body to the generated short-note
overlay, and include it in README discovery and validation before `AGENTS.md` is
composed. Planning, `--check`, `--diff`, application, git-state classification,
idempotence, invalid-config failure, generated-note retirement, and provider-shim tests
must cover this ordering. Home memory initialization must never consume a user/global
`glossary` value.

## Shared catalog and source navigation

Implement the canonical domain in `sase-core`: typed entry/catalog/span wires,
normalization and collision diagnostics, a compiled longest-match scanner,
lookup-at-position, and deterministic Markdown-ready data. Expose a frozen PyO3
matcher/catalog handle so ACE can scan from memory on every repaint without reparsing
YAML or recompiling patterns. Keep source-preserving YAML parsing and line/column
extraction in Python, using the existing ruamel infrastructure, and attach the
definition scalar's source range plus canonical config path to each entry. Rust owns
semantics; Python owns file discovery, source locations, and presentation.

Add a versioned, read-only editor glossary-catalog helper contract. It resolves an
explicit project name/key/alias to an enabled project's primary workspace, loads only
that workspace's project-local config, and returns normalized entries and source
locations. ACE caches compiled catalogs per canonical project and a config
`(path, mtime, size)` signature, warms/reloads them in background workers, and exposes a
memory-only exact-project getter to prompt widgets. The LSP uses the same helper bridge
and a bounded per-project cache with timeout and stale fallback behavior. Config writes,
project changes, the existing explicit LSP catalog refresh, and watched `sase.yml`
changes invalidate the relevant cache; the LSP requests a semantic-token refresh after
invalidation. No disk reads, subprocesses, config resolution, or regex compilation may
occur in ACE render or key-dispatch paths.

Because the public Rust/PyO3 and helper-bridge contracts change, land and release the
`sase-core` work before raising the main repository's `sase-core-rs` dependency floor
and lockfile to the first released version containing those APIs. Let release-plz choose
core crate versions; do not edit core version fields manually. Local development and
verification should build the binding and LSP from the linked `sase-core` checkout.

## ACE experience

Add a glossary overlay late enough in `PromptTextArea`'s mixin chain to win over plain
Markdown but not over explicit xprompt/directive, artifact-reference, code-literal,
selection, search, or diagnostic overlays. Highlight the full matched phrase with a
theme-derived semantic-type treatment: a readable primary or cyan-like color with modest
emphasis, never an underline (reserved for problems/links) and never warning/error
colors. Register it through the existing light/dark TextArea theme composition and keep
the current prompt size guards. Add unit, theme-contrast, performance, and PNG snapshot
coverage so the result is distinct yet calm in both themes.

Preserve existing action precedence. `K` continues to preview explicit xprompts, skills,
and files first; on ordinary text it checks the warm glossary match under the cursor
before falling back to spellcheck/dictionary lookup. The glossary preview reuses the
rendered Markdown preview surface and shows the canonical term, definition, optional
configured aliases, project, and source. `Ctrl+]` resolves the same match to the project
config's `definition` range and uses the established editor/tmux jump flow. If a catalog
is cold, schedule the warm and show a concise retry notice; if context changes while
work is running, discard the stale result. Preserve current behavior for counted keys,
missing matches, malformed catalogs, and non-glossary words.

## LSP experience

Extend the semantic-token legend without renumbering existing artifact token types. Emit
a standard `type` semantic token for each glossary phrase so editors receive a familiar
type/concept color without client-specific configuration. Use the shared matcher and the
same project selection, code-literal exclusions, longest-match rules, UTF-16 conversion,
and line-local constraint as ACE. Merge and sort glossary and artifact tokens before
delta encoding, ensuring the streams never overlap or produce invalid deltas.

Glossary-aware hover returns the definition as Markdown with canonical term, aliases,
and project/source context. Go-to-definition returns the project-local `sase.yml`
definition range, which gives external editors their conventional `K` and `Ctrl+]`
behavior. Explicit xprompt/reference hovers and definitions keep precedence over
glossary matches. Cover initialize capabilities, light clients with no semantic-token
support, leading VCS project aliases, active-workspace fallback, multi-project
documents, watched-config refresh, Unicode/UTF-16 spans, code literals, overlap
resolution, catalog failure fallback, hover, definition, and JSON-RPC stdio behavior.

## Migration, documentation, and acceptance

Move the current SASE glossary content into `sase/sase.yml`, preserving all definitions
while making canonical entries useful to matching. Keep established terms; normalize the
parenthetical `Agent Instruction Files (aka agents.md files)` into a canonical
`Agent Instruction Files` term with explicit aliases, and split the combined
`Projects, Repos, and Workspaces` section into `Project`, `Repo`, and `Workspace`
entries so each concept can be highlighted and opened without making an enormous phrase
the only lookup key. Add conservative singular/plural aliases for the remaining terms,
avoiding noisy abbreviations or generic aliases that do not clearly denote the SASE
concept.

As a controlled migration for this repository, remove the old unmarked manual
`sase/memory/glossary.md`, run `sase memory init`, and inspect the regenerated glossary,
README, `AGENTS.md`, and provider shims. The user's prompt grants the required
permission for these canonical memory and generated instruction-file changes. Document
the config shape, project scoping, generation command, generated-file ownership,
matching semantics, highlighting, `K`, `Ctrl+]`, LSP hover/definition behavior, and how
to resolve validation collisions.

Acceptance requires all of the following:

- One init invocation changes a config definition into matching generated glossary text
  and the inlined agent instructions; a second invocation is a no-op and
  `sase memory init --check` passes.
- Effective retrieval includes each term exactly once even when `aliases` is omitted,
  while ambiguous aliases fail consistently in schema/domain/init, ACE, and LSP tests.
- Switching only the leading VCS workflow reference switches highlighted, previewed, and
  edited glossary entries in ACE and LSP without restarting.
- ACE typing/highlight paths remain disk- and subprocess-free and retain the
  repository's prompt responsiveness budgets.
- `K` renders the right project definition and `Ctrl+]` opens the correct project-local
  config range; external LSP hover and go-to-definition return equivalent
  content/locations.
- Existing xprompt, artifact, code-block, misspelling, search, and jump behavior remains
  covered and unchanged outside glossary matches.

Before landing, run the `sase-core` workspace formatter check, Clippy with warnings
denied, and full Rust workspace tests; then rebuild/install the local binding, run the
main repository's `just check-full`, dedicated ACE visual snapshot suite, LSP JSON-RPC
tests, schema/config tests, memory-init suites, and `sase memory init --check`. Inspect
the final diffs in both repositories and confirm no release-plz-owned version field was
hand-edited.
