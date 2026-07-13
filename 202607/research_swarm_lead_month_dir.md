---
create_time: 2026-07-13 09:07:58
status: done
prompt: 202607/prompts/research_swarm_lead_month_dir.md
tier: tale
---

# Plan: Research Swarm Lead Segment — Month Directory, Lead-Researcher Role, and Concision

## Goal

Revise the third (final/consolidator) segment of the chezmoi-managed `research_swarm` xprompt so that:

1. The final agent creates its `<name>/` output directory inside the current-month research directory, which is handed
   to it explicitly via xprompt shell expansion rather than left implicit.
2. The final agent understands it is the **lead researcher**: it must perform its own research on the request and merge
   those findings — together with the two source reports — into the consolidated report.
3. The segment is rewritten with concise, high-signal prompting. Every retained sentence must earn its tokens.

## Current Behavior

The active xprompt lives at `~/.xprompts/research_swarm.md`, deployed by chezmoi from the managed source
`home/dot_xprompts/research_swarm.md` in the linked `chezmoi` repository. Its third segment currently:

- Computes the "effective research directory" with `$(sase sdd path research --ensure)`, which resolves to the research
  repo **root**. The consolidator therefore builds `<root>/<name>/` while both independent researchers write their
  reports under `<root>/<YYYYMM>/` (their `#research` config xprompt already appends `$(date +%Y%m)`), and the research
  repo README documents the `<YYYYMM>/<topic>/` layout. The consolidator is the odd one out.
- Frames the final agent purely as a verifier/consolidator; it performs no research of its own.
- Spends ~340 words on instructions that can be stated in roughly half that.

## Design Decisions

### Month directory format: `$(date +%Y%m)`, not `date +%y%%m`

The request suggested passing the month directory with `date +%y%%m`. This plan deliberately uses `$(date +%Y%m)`
instead, for two verified reasons:

1. **No `%%` escape layer exists.** Xprompt command substitution (`process_command_substitution` in
   `src/sase/file_references.py`, invoked from `preprocess_prompt_late`) passes the command verbatim to `sh -c`. The
   directive stripper cannot match `%Y`/`%m` here because `_DIRECTIVE_PATTERN` requires `%` to appear at line start,
   after whitespace, or after `([{"'` — inside `date +%Y%m` neither `%` qualifies. So no escaping is needed, and a
   literal `date +%y%%m` would actually print `26%m` (the shell's `date` renders `%%` as a literal percent sign).
2. **Convention consistency.** The two researcher agents already write to
   `$(sase sdd path research --ensure)/$(date +%Y%m)/` via the `#research` config xprompt, the research repo contains
   only `<YYYYMM>` month directories (`202602` … `202607`), and its README documents the `<YYYYMM>/<topic>/` layout. A
   prior completed plan (`dynamic_research_xprompt_paths`) also confirmed this exact `$(date +%Y%m)` substitution works
   unescaped in this file. Using `%y%m` (`2607`) would fork the on-disk convention and strand the consolidated report in
   a different directory than the source reports.

If the two-digit-year form is genuinely wanted despite the above, reject this plan and say so; the change is a
two-character edit to the proposed text.

### Lead's research goes into the consolidated report, not a third source file

The lead's own findings merge directly into `<name>.md`. No `<name>__lead.md` is added: the `__a`/`__b` files exist to
preserve the independent inputs, while the lead's contribution _is_ the final report. This keeps the documented
three-file layout stable for downstream consumers (the image segment and `#research/more`).

### Month rollover is acceptable

If a swarm straddles a month boundary, the source reports are found via the transcripts (authoritative paths) and
relocated into the _current_ month directory. This is accepted behavior and needs no extra prompt text.

## Implementation

All edits target the linked `chezmoi` repository, opened with the required
`sase workspace open -p chezmoi -r "<reason>" <workspace_num>` workflow. The only file changed is
`home/dot_xprompts/research_swarm.md`.

1. Optionally sharpen the frontmatter description to mention the lead role, e.g. "Launch two independent research
   agents, then have a lead researcher extend and consolidate their findings and generate an infographic."
2. Leave segments one (`research.@.cdx`), two (`research.@.cld`), and four (`research.@.image`) unchanged, including all
   model aliases, waits, and group directives.
3. Replace the body of the third segment (everything after its directive line, which is also kept unchanged) with:

````markdown
%name:research.@.final %m:@research_lead %wait:research.@.cdx %wait:research.@.cld %g:research

You are the lead researcher: two independent researchers have reported on the request below, and you will add your own
research and merge all three perspectives into one consolidated report.

Research request:

{{ prompt }}

The researchers' chat transcripts:

{% raw %}{{ wait_chats }}{% endraw %}

Month directory (create it if missing):

$(sase sdd path research --ensure)/$(date +%Y%m)

Steps:

1. Read both transcripts to learn which report file each researcher wrote (`research.@.cdx` -> `__a`, `research.@.cld`
   -> `__b`), then read both reports. Never assign `__a`/`__b` from filesystem order.
2. Research the request yourself, prioritizing gaps, weak evidence, and disagreements between the two reports.
3. Pick a descriptive stem `<name>` that collides with nothing in the month directory, create `<month-dir>/<name>/`, and
   move the two reports to `<name>__a.md` and `<name>__b.md` inside it. Preserve both files and never overwrite: on any
   collision, pick a different stem first.
4. Write the consolidated report to `<name>/<name>.md`: merge the strongest findings from both reports and your own
   research, resolve conflicts, cut duplication, and add missing critical context without unnecessary length.

Final layout:

```text
<month-dir>/<name>/
├── <name>__a.md
├── <name>__b.md
└── <name>.md
```
````

Notes on the rewrite, relative to the current text:

- The month directory is now handed over explicitly and the agent is told to create it if missing (`--ensure` only
  materializes the research repo clone, not the month subdirectory).
- The lead-researcher role statement opens the prompt, and step 2 makes the independent-research obligation explicit and
  targeted (gaps, weak evidence, disagreements) rather than open-ended.
- The deterministic `__a`/`__b` producer mapping, the no-silent-overwrite rule, the preserve-both-sources rule, and the
  no-length-bloat consolidation requirements are all retained, just stated once each.
- The Jinja `{% raw %}` guard around `wait_chats` and the `{{ prompt }}` placeholder are preserved exactly as today.

4. Deploy by applying only this chezmoi target so `~/.xprompts/research_swarm.md` matches the managed source, and
   confirm a subsequent targeted `chezmoi diff` is empty.

## Validation

1. `sase xprompt expand '#research_swarm:: test'` — confirm the expansion still yields a four-agent markdown swarm; that
   the third segment contains the resolved `<research-root>/<YYYYMM>` path for the current month; and that the raw
   `wait_chats` placeholder survives unrendered.
2. Grep the rendered third segment to confirm no stale root-level "effective research directory" instruction remains and
   that `26%m`/`%y%m` never appears.
3. Run the chezmoi repository's formatting/structural checks for the edited markdown (and `git diff --check`).
4. Inspect the final chezmoi diff to confirm only `home/dot_xprompts/research_swarm.md` changed — segments one, two, and
   four byte-identical apart from the optional frontmatter description tweak.
5. No SASE `just check` gate applies: the change is personal chezmoi markdown only, with no SASE-repo file changes (same
   precedent as the completed `rename_research_c_to_research_lead` plan).

## Non-Goals

- Do not change the researcher segments' prompts, models, waits, or groups, nor the image segment.
- Do not modify the `research`, `research/more`, `research/prompt`, or `research/image` config xprompts in the managed
  `sase.yml`.
- Do not touch the legacy `old_research_swarm.md` xprompt.
- Do not change SASE source code or backend behavior — this is a prompt-contract change in personal chezmoi sources.
