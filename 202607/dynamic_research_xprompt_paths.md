---
create_time: 2026-07-09 02:30:03
status: done
prompt: 202607/prompts/dynamic_research_xprompt_paths.md
tier: tale
---
# Plan: Dynamic Research XPrompt SDD Paths

## Context

Some research-oriented xprompts still assume that the canonical research directory is `sdd/research/`. That is wrong for
workspaces using a separate SDD store, where the effective directory can be `.sase/sdd/research` or another resolved
store path.

The live `#research` expansion currently comes from the chezmoi-managed SASE config, not from in-repo built-in xprompt
files:

- `/home/bryan/.local/share/chezmoi/home/dot_config/sase/sase.yml`
- `/home/bryan/.local/share/chezmoi/home/dot_xprompts/research_swarm.md`

`sase sdd path research` is the right resolver because it prints the effective canonical research child directory for
the current workspace. Xprompt command substitution already works here, as shown by the existing `$(date +%Y%m)`
expansion.

## Scope

1. Update the `research` config xprompt in the chezmoi source config so it writes to:
   - `$(sase sdd path research)/$(date +%Y%m)/`
   - instead of `sdd/research/$(date +%Y%m)/`
2. Update `research/more` in the same config so its README convention reference points at the resolved research
   directory:
   - `$(sase sdd path research)/README.md`
3. Update the chezmoi source `research_swarm.md` so the final consolidator:
   - computes the effective research directory with `$(sase sdd path research)`;
   - identifies and deletes intermediate files in that resolved directory;
   - writes the final consolidated file in that resolved directory rather than saying `sdd/research/`.
4. Do not edit live generated/configured outputs directly unless needed only to verify deployment. The source of truth
   is the chezmoi repo.

## Validation

1. Search the xprompt sources again for stale unconditional `sdd/research/` write instructions.
2. Run `sase xprompt expand '#research test'` and confirm it expands to the effective research path for this workspace.
3. Run `sase xprompt expand '#research_swarm:: test'` and confirm the final consolidator prompt references the resolved
   research directory.
4. Apply the chezmoi changes if needed so the live config matches source, then re-run the xprompt expansion checks.
5. Because this run creates a plan file in the SASE repo, run the repo's required check flow after edits: `just install`
   if needed, then `just check`.
