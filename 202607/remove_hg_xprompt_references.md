---
create_time: 2026-07-10 10:24:46
status: done
prompt: 202607/prompts/remove_hg_xprompt_references.md
tier: tale
---
# Remove obsolete `#hg` xprompt references

## Goal

Remove the retired `#hg` workspace-xprompt workflow from SASE's active surface so core no longer documents, recognizes,
or synthesizes it when no workspace provider registers it. Preserve the provider-neutral workspace architecture: a
future plugin must still be able to register any workflow name through `WorkflowMetadata` without core code knowing the
name in advance.

The repository audit found literal `#hg` references across user documentation, runtime comments/docstrings, generated
skill source, and tests, plus less-visible behavioral assumptions: `hg` remains in prompt fallback/alias allowlists,
agent-workflow prefix matching, and the default `vcs_type` for CRS, mentor, fix-hook, and mentor-change workflows. Those
implicit paths can still classify or emit `#hg` even if all literal examples are removed.

## Scope boundary

- Remove references and hard-coded behavior specifically associated with the retired `#hg` workspace xprompt.
- Retain generic VCS/workspace plugin APIs and tests that validate arbitrary registered providers.
- Retain low-level Mercurial compatibility code, provider contracts, diff parsing, and generic Mercurial documentation
  that do not advertise or assume the `#hg` xprompt. Those are a distinct provider-layer concern.
- Retain the separately documented, Gemini-only `sase_hg_commit` skill. The audited generated-skills memory currently
  defines that as an intentional commit-finalizer contract, not as the retired workspace xprompt. No protected memory
  files need to change for this cleanup.
- Do not regenerate image assets: visual inspection/OCR found no `#hg` token in the tracked documentation images. Text
  critiques that still propose the token will be corrected.

## Implementation plan

### 1. Remove core fallback recognition of the retired workflow

- Delete `hg` from prompt/workspace fallback sets in the xprompt parser, cheap xprompt-reference detector, project-alias
  canonicalizer, and prompt-history metadata paths. Registered workspace metadata remains the authoritative way for a
  plugin workflow to participate in parsing, completion, history, aliases, and ref resolution.
- Replace the hard-coded `hg-` agent-workspace prefix in ACE deduplication with registered workflow metadata (or an
  equivalent provider-neutral predicate), so duplicate embedded workspace claims are still removed for Git/GitHub and
  arbitrary plugins without retaining a legacy Hg special case.
- Generalize nearby docstrings, examples, and comments in ACE, axe, query handling, xprompt parsing/execution, and
  workflow models to use supported `#git`/`#gh` examples or provider-neutral terminology.

### 2. Stop workflow builders from synthesizing `#hg`

- Remove the `hg` defaults from the CRS and mentor prompt builders and from the `mentor`, `make_mentor_changes`, and
  `fix_hook` xprompt inputs. Require the caller to supply the detected workspace workflow whenever a ChangeSpec ref is
  embedded, while allowing ref-free invocations to remain ref-free.
- Update every live caller to obtain the workflow type through the workspace-provider registry. In particular, fix the
  older ACE CRS path, which currently passes a ChangeSpec name without a workflow type and therefore falls back to
  `#hg`; axe CRS, mentor, fix-hook, and mentor-change paths already provide a model for provider-derived dispatch.
- Add validation or focused tests at the prompt-builder boundary so a ChangeSpec-bearing invocation cannot silently
  choose a retired/default provider.

### 3. Refresh active documentation and generated guidance

- Replace literal `#hg` examples in the workspace/xprompt/mentor guides, relevant blog posts, integration examples, and
  the xprompt-resolution critique with current `#git`/`#gh` examples or explicit “registered provider” wording.
- Update `src/sase/xprompts/skills/sase_run.md` so agent-launch guidance lists the supported Git/GitHub refs and
  explains that additional refs come from installed workspace plugins, without naming the retired workflow.
- After the source update and local editable install, run `sase skill init --force` and `chezmoi apply` as required by
  the generated-skills contract; inspect the generated skill diff to ensure only the intended guidance changed.

### 4. Convert legacy Hg fixtures into provider-neutral coverage

- In xprompt parsing, replacement, completion, MRU, disabled-region, wraps-all, embedded-output, project-local, and TUI
  execution tests, use `#git`/`#gh` when the provider identity is irrelevant and a deliberately fake registered workflow
  (for example `spy`) when the test must prove plugin extensibility.
- In ChangeSpec-tag, workspace-claim, agent deduplication, bare-git safety, and workspace-provider utility tests,
  replace `hg`/`hg-*` fixture values with the same neutral registered-provider fixture. Update shared
  metadata/cache-reset helpers rather than duplicating one-off regexes.
- Remove `hg` from expected fallback workflow catalogs and alias sets. Keep tests whose subject is genuinely low-level
  Mercurial/provider behavior unchanged.
- Add a focused regression test, analogous to the removed-directory-workspace coverage, proving that an unregistered
  `#hg` ref is not exposed as a core workflow or treated as a fallback workspace ref, while a dynamically registered
  fake provider continues to work through the same generic paths.

### 5. Verify the removal comprehensively

- Re-run case-insensitive tracked-text searches for `#hg`, Hg/xprompt or Hg/workspace-workflow phrasing, hard-coded
  fallback sets/defaults, and obsolete `sase-google` identities. Classify any remaining `hg`/Mercurial hits and confirm
  each is genuinely provider-layer behavior rather than an xprompt assumption.
- Check the rendered xprompt/skill catalog and generated skill locations to ensure active guidance no longer exposes
  `#hg` and no generated copy reintroduces it.
- Run focused tests for xprompt parsing/replacement/completion, prompt metadata/project aliases, CRS/mentor/fix-hook
  prompt construction, ACE agent deduplication, ChangeSpec tags, and skill generation.
- Run `just install` before repository commands as required for an ephemeral workspace, then finish with `just check`.
  Review both the SASE and generated chezmoi diffs so no unrelated user changes are included.

## Acceptance criteria

- No tracked source, test, documentation, or generated skill guidance contains a reference to the retired `#hg`
  workspace xprompt.
- Core does not recognize, normalize, complete, summarize, alias-rewrite, or synthesize `hg` as a workspace workflow
  unless a workspace plugin explicitly registers that name.
- CRS, mentor, fix-hook, and mentor-change launches use the workflow type detected for their project and never fall back
  to Hg.
- Generic workspace-plugin behavior remains covered with a neutral fake provider, and unrelated provider-level Mercurial
  behavior remains intact.
- Focused tests and the full `just check` suite pass.
