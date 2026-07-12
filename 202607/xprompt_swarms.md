---
create_time: 2026-07-08 20:45:50
status: done
prompt: 202607/prompts/xprompt_swarms.md
tier: tale
---
# Rename multi-agent xprompts to xprompt swarms

## Goal

Audit the main SASE repo, `sase-core`, and the SASE plugin repos for references to the current concept name "multi-agent
xprompt" and update the affected prose and code-facing terminology to the new term "xprompt swarm" / "xprompt swarms".

The requested canonical glossary update is in scope: update `memory/glossary.md`. Because the generated provider
instruction shims and `AGENTS.md` currently contain the same glossary entry, treat those exact glossary mirrors as in
scope too, while avoiding unrelated memory or instruction churn.

## Scope

Repos to audit:

- `sase`
- `sase-core`
- `sase-github`
- `sase-telegram`
- `sase-nvim`

Initial searches show direct old-term hits are concentrated in `sase`. The linked repos need a verification pass, but
currently do not appear to contain direct `multi-agent xprompt` wording.

Affected clusters in `sase`:

- Canonical and generated glossary text: `memory/glossary.md`, `AGENTS.md`, provider shims.
- User docs: `docs/xprompt.md`, `docs/ace.md`, `docs/configuration.md`, README/blog/demo text where the phrase refers to
  xprompt-defined fan-out.
- Code comments, docstrings, and user-visible errors around dispatch-time expansion, prompt stack integration, and
  xprompt catalog handling.
- Tests and fixtures whose names/docstrings/comments encode the old term.

Do not mechanically replace every `multi-agent prompt` occurrence. Some references describe literal user prompts or
prompt-stack behavior that is broader than xprompt swarms and should remain as-is unless the surrounding text is clearly
about xprompt-defined fan-out.

## Implementation plan

1. Build the complete hit list with multiple searches across all repos:
   - Direct old terms: `multi-agent xprompt`, `multi_agent_xprompt`, `multi agent xprompt`.
   - Nearby concept terms: `multi-agent prompt`, `multi_agent_prompt`, `segment separators`, `--- multi-agent`,
     `multi-agent.*---`.
   - New-term guard: `xprompt swarm`, `xprompt swarms`.

2. Classify each hit before editing:
   - Rename xprompt-specific concept text to "xprompt swarm(s)".
   - Leave generic launch, prompt-stack, multi-model, repeat-agent, and persisted multi-prompt file terminology alone
     unless it is specifically referring to library-defined xprompt fan-out.
   - Prefer natural prose over literal substitutions, especially in headings and glossary definitions.

3. Update canonical docs and glossary:
   - Change the glossary entry heading from "Multi-agent xprompt" to "Xprompt swarm".
   - Describe an xprompt swarm as an xprompt whose body contains top-level `---` separators outside fenced blocks and
     fans out into one agent per segment at launch.
   - Update docs section names, anchors, cross-links, and nearby explanatory text for xprompt swarms.

4. Update code-facing terminology where it names this xprompt-specific concept:
   - Update comments, docstrings, errors, and test descriptions.
   - Consider renaming internal Python module/test/helper identifiers from `multi_agent_xprompt` to `xprompt_swarm` if
     the hit list shows those identifiers are local and not part of a documented public API. If a compatibility risk is
     found, keep behavior/API stable and limit the change to prose/user-facing strings.

5. Re-run the audit searches:
   - Confirm no stale `multi-agent xprompt` / `multi_agent_xprompt` references remain except any intentionally retained
     compatibility shim, if one is needed.
   - Inspect remaining `multi-agent prompt` hits and document why they are not xprompt swarm references.
   - Confirm linked repos remain clean or apply any necessary local updates there.

6. Verify:
   - In the main `sase` repo, run `just install` and then `just check` because this repo requires that after file
     changes.
   - If any linked repo is edited, run that repo's local checks or, if unclear, at minimum its focused test/lint target
     discovered from its project files.

## Expected outcome

The codebase and linked repos consistently use "xprompt swarm" / "xprompt swarms" for the former multi-agent xprompt
concept, with unrelated generic multi-agent launch terminology preserved.
