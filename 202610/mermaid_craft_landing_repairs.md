---
tier: tale
title: Finish Mermaid craft contracts and land sase-1nd
goal:
  The shared Mermaid skill is usable by the lead and diagram producer, all original
  acceptance checks pass, and epic sase-1nd is closed.
size: medium
proposed_by: bbugyi200.athena.sase-1nd.land
proposed_with: codex/gpt-6-astra
bead: sase-1nd
create_time: 2026-10-20 13:31:02
status: wip
---

- **PARENT:**
  [202610/research_mermaid_diagram_craft.md](https://github.com/sase-org/sase--plans/blob/main/202610/research_mermaid_diagram_craft.md)
- **BEAD:**
  [sase-1nd](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1nd/README.md)

# Finish the Mermaid craft contract and land sase-1nd

## Goal and scope

Finish the remaining implementation gaps in epic `sase-1nd`, then close that epic in
this coding turn. Its two phases are already closed, but the landing audit found that
the shared skill cannot yet carry out the approved render recipe and does not reach all
lead-authored Mermaid figures. This is one bounded plugin repair, not another
multi-phase epic.

Read `sase bead read sase-1nd -r "Need landing audit and final decisions"` and both
children (`sase-1nd.1`, `sase-1nd.2`) with audited reasons. Read these artifacts:

- `plan:202610/research_mermaid_diagram_craft.md`
- `research:202610/research_swarm_mermaid_diagram_craft/research_swarm_mermaid_diagram_craft.md`

Use `sase artifact read <ref> "<reason>"` for both. If the research reference is
missing, open/refresh `research` with `/sase_repo`, then retry the audited read. The
accepted reviewer decision is **lead_reach = lead**: give the swarm lead on-demand
guidance; do not add it to standalone `research.md` or researcher segments, and do not
inline the entire skill into the lead.

Open `sase-research-artifacts` with `/sase_repo` and read its `AGENTS.md`. All
implementation changes belong there. The only other expected file change is the epic
plan's status in the plans sidecar during closeout. Do not change the renderer repo,
deploy global skills, install globally, or modify release-owned CHANGELOG.md. Plugin
text must pass `tests/test_no_bob_dependency.py`; use renderer-generic names.

## Audit evidence already collected

The lander reviewed all child notes, the accepted plan and research, and plugin commits
`b16687b` (skill) and `af3a591` (producer). The candidate ranking, fidelity rules, 3/5
caps, manifest placement and audit fields, fail-open behavior, packaged skill/SVG
resources, and single inlined guide are present. Preserve them.

At audit time plugin HEAD and origin/master were `af3a591`, with no non-epic commits
since `b16687b`. Fetched SASE origin/master was `9188cc5733`; the only host change since
epic creation split unrelated deck-title tests. No integration change was needed.
Recheck subsequent drift before closing.

The epic has no `parent_bead`. `sase bead epic-symbols sase-1nd` returned no entries.
Recheck both at closeout rather than relying on this snapshot.

All proposed follow-ups from `sase-1nd.2` were triaged and recorded on `sase-1nd`:

- Note #1: corroborated existing ready task `sase-1mo` with +1. Its post-release pilot
  now includes an early 5–10-swarm checkpoint, title/question coverage,
  > =80% full-size figures and none below 75%, blind ratings, omissions, chart-type mix,
  > render cost, version cohorts, and measured reasons to split the segment.
- Note #2: declined optional deterministic lint for now. The proposal depends on a noisy
  look loop, and no post-change evidence establishes that condition.
- Note #3: corroborated existing ready task `sase-1mn` with +1 for actual-renderer
  parity. Its note clarifies that the old vault-dependent producer is historical:
  current production uses mmdc. Declined the separate optional house theme pending
  measured need and a renderer-owner decision.

No new task was created. Do not duplicate this triage; include these outcomes in the
close note.

## 1. Make the shared skill's render recipe complete

Edit `src/sase_research_artifacts/skills/mermaid_diagrams.md`. It currently states the
fit formula in prose but its measurement command only prints width/height, and it
chooses global mmdc before an unpinned npx fallback. Replace that partial recipe with
the approved, runnable recipe:

- Create a short scratch directory with `mktemp -d /tmp/mmd.XXXXXX` and export that path
  as `TMPDIR`.
- Write the existing PDF-mirroring JSON inline to `pdf.json`; preserve the strict
  security setting, locked neutral theme and secure-key list, font, and
  `useMaxWidth: false` settings.
- Select the CLI once: smoke-test `npx -y -p @mermaid-js/mermaid-cli@12.0.0 mmdc` first,
  then global `mmdc`. Success requires exit zero and the expected SVG. Preserve the one
  sandbox-start retry with Puppeteer's `{"args":["--no-sandbox"]}` and reuse the chosen
  command and flags thereafter. Record the version. A browser-start failure is an
  environment failure, not evidence that a diagram is bad.
- Render SVG and PNG with the same `-c <scratch>/pdf.json`; PNG uses `-s 1.5 -b white`,
  a bounded timeout, and one retry for a Puppeteer connection failure.
- Supply one small verified Python measurement command in the skill. Parse the root
  SVG's numeric pixel width/height, falling back to the root `viewBox` dimensions for
  nonnumeric/missing dimensions. Reject unusable/nonpositive dimensions. Print
  `W×H px, fit N%` using `min(1, 652.8 / W, 892.8 / H)` and make below-0.75 results
  visibly fail so callers cannot mistake them for passing figures. Never inspect an
  internal max-width.
- Open the PNG and answer the reader question, check labels, arrowheads, crossings,
  reading direction, and fit, and trace entities/relations to source. State the original
  plan's fallback for runtimes without image viewing: disclose it and rely on measured
  and source checks. Allow two revisions, then drop.

Keep general rendering knowledge in this shared skill. Reconcile
`xprompts/research_diagrams.md` step 5 so the same procedure is used by the producer;
avoid competing CLI-selection recipes. Preserve the batch check with
`-c <scratch>/pdf.json`, `Found N mermaid charts`, N output SVGs, five parse-fix rounds
distinct from two visual revision rounds, and fail-open status handling.

## 2. Restore omitted craft details and repair lead access

Reconcile the skill with the original plan's explicit details, using positive rules with
short reasons rather than expanding its scope:

- Use the approved fixed palette: focus `#E8F1FB/#3B78C2/#0F2E52`, decision
  `#FFF7E6/#C77C02/#4A3000`, outcome `#E7F5EC/#2E7D4F/#123D24`, risk
  `#FDECEC/#C0392B/#5F1414`, and muted `#F3F4F6/#6B7280/#1F2937` (fill/stroke/text).
  Restore the `proposed` dash modifier `stroke-dasharray:5 3`, stacked with a color
  class through two `class` lines. Keep at most three semantic color classes in each
  figure.
- Explain real mode/owner/phase subgraphs with `fill:none` and their class's stroke; use
  node-to-node edges where arrowheads matter.
- Restore quoted labels, avoiding `end` node IDs, optional invisible
  `accTitle`/`accDescr`, about five-word decision questions, and `wrappingWidth: 260` as
  a tightening option.
- Keep the established family menu; explicitly exclude the original plan's
  timeline/pie/mindmap/new-shape families. Quadrants still require source-backed
  axes/coordinates. Do not invent structure or mandatory start/end nodes.

Bring example sources, their stored SVGs, and dimensions into agreement with this recipe
and palette. The current mode example drops the locked-spread state from the reference
design: use a faithful compact example covering the two automatic layouts and the two
locked layouts, distinguishing the proposed toggle where shown, with visible node-level
return arrowheads. Use the original report's source statements and keep proposed
behavior marked. The routing example must remain source-faithful. Prefer 100% fit; all
examples must meet the 75% floor.

In `xprompts/research_swarm.md`, remove the `run_diagrams` restriction on the lead
pointer. A lead that draws or carries over Mermaid needs the skill even when
`image=false, diagrams=false`; this does not enable a companion diagrams agent. Add the
original plan's fallback command for an undeployed provider skill: verify
`sase macro show skill/mermaid_diagrams --format raw` (or the slash form) against the
edited plugin and use the form that works. Keep this command as literal prompt text; do
not introduce a macro reference that inlines the guide. Researchers and standalone
`research.md` remain untouched.

## 3. Complete coverage, documentation, and render acceptance

Update the plugin's tests with meaningful coverage of the missing contract:

- Exercise the measurement snippet on representative flowchart, state, and sequence SVG
  dimensions, including viewBox fallback and a below-floor case. Test the actual shipped
  command/snippet rather than a separately copied implementation. Real Mermaid renders
  below supply integration evidence.
- Keep the existing inline macro expansion test, and add a generated-SKILL round trip
  through the actual host renderer or a read-only `sase skill init` preview against the
  edited plugin. The host exposes `sase.main.init_skills_handler.render_skill_targets`,
  a rendering API that writes no files. Check preserved hexagons, frontmatter, hex
  colors, and absent raw tags. Respect the plugin's published SASE floor through its
  compat helpers.
- Pin the skill's executable recipe and revision/fit contracts, not just the producer's
  duplicate wording. Test the lead pointer and fallback with diagrams on and off, no
  inlined skill in the lead, no researcher pointer, exactly one inlined guide in the
  diagrams-on image segment, unchanged diagrams-off behavior, the nine-segment contract,
  and audit fields confined to the manifest.
- Preserve wheel/sdist resource tests and fresh-wheel skill discovery.

Update README, docs/architecture.md, docs/macros.md, and the AGENTS.md architecture
bullet to match: skill source/discovery, inline and on-demand usage, lead access
independent of companion flags, the fallback, home-scope overrides, and the complete
render/fit procedure. Remove the stale hard "half a page" wording in docs/macros.md in
favor of the approved soft 650×700 target and 75% fit floor. Keep mirrored instruction
files consistent with the repo's convention. Read required documentation/skill memories
before editing these files.

Run `sase tool run check` from the plugin repo (the guarded `just check` recipe). Run
`just test-wheel` for the changed packaged resources, using the repository's supported
source setup and opening any needed linked dependency with `/sase_repo` first. Do not
run `just check-full`.

Render all final worked examples and the `publication-order` manifest example using
pinned mmdc and the skill's exact recipe. Check a flowchart, a state diagram, and a
sequence diagram with the measurement command; verify every color class and the stacked
proposed modifier; verify a frontmatter theme override remains neutral and deliberately
broken Mermaid exits nonzero. View the PNGs when the runtime has an image-viewing tool.
Record dimensions/fit, rounds, version, visual findings, and any actual viewing
limitation. Keep the publication-order example at 100% fit, and synchronize its
source/docs/metadata after edits. Do not make font/browser-dependent pixel goldens.

If commands require a monitor, use `/sase_monitor` with a continuation that includes
this plan's remaining closeout. Do not end a turn with an unobserved running command. Do
not wait for this work's own commit, push, SHA, or CI run: the host commits after the
coding turn ends.

## 4. Finish sase-1nd's landing in this same coding turn

There is no subsequent land agent for this tale. Once the fixes and checks pass:

1. Reread `sase-1nd`, both children and their notes, the original plan, and this repair
   plan. Verify all required work is complete, and review plugin and host changes since
   the audit SHAs above for new integration needs.
2. Run `sase bead epic-symbols sase-1nd`. Resolve each returned symbol by wiring,
   privatizing, adding an allowed non-test pragma, or deleting it per the Symvision
   policy. Re-key a Justfile exemption only if a named still-open later bead needs it.
   Read the Symvision memory before such edits. Leave no exemption keyed to this epic or
   its closing phases.
3. Close normally with `sase bead close sase-1nd --note "<verification>"`. The concrete
   note must summarize the repaired recipe and lead access, palette/example/round-trip
   coverage, actual check and render outcomes, integration review, and the follow-up
   outcomes: `sase-1mo` and `sase-1mn` corroborated; deterministic lint and house theme
   declined for lack of the triggering evidence. If exemptions block closing, finish
   them and retry. If unfinished descendants block it, finish/reopen the required work.
   Never force to make closure succeed or to advance a successful nested landing.
4. After close, run `just symvision` from the SASE host checkout, where that recipe
   exists, and confirm the whitelist is clean.
5. Use `/sase_repo` to open `plans`. Read
   `plan:202610/research_mermaid_diagram_craft.md` with `sase artifact read`, then set
   only its frontmatter `status: done` in `202610/research_mermaid_diagram_craft.md`
   inside the printed plans checkout.
6. Run `sase bead read sase-1nd -r "Need the parent link"` after closing. The audited
   epic has no parent, so finish normally. If that relationship changed, do not guess or
   force-close an unfamiliar parent: record the ambiguity on it and report it.
7. Use `/sase_final` as the final action of a normal turn, declaring commits for every
   edited repository, including the plugin and plans sidecar. The closeout and plan
   status update happen before the host creates these commits.
