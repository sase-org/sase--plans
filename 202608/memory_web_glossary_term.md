---
tier: tale
title: Refresh the Memory Web glossary strands
goal:
  The Memory Web, Memory Strand, and Strand Keyword glossary strands describe the
  shipped web substrate accurately and concisely, with no "Not yet implemented" claim
  left anywhere under sase/memory/ and no growth in the mention closure a
  glossary:memory-web read pays for.
size: small
proposed_by: bbugyi200.athena.0dl
---

# Refresh the Memory Web glossary strands

## Problem

The `Memory Web`, `Memory Strand`, and `Strand Keyword` glossary strands all still end
with "Not yet implemented." That was true when the vocabulary was reserved ahead of the
substrate; it stopped being true when epic `sase-sq` ("Memory webs and strands") closed
`done` on 2026-08-25 with all eight phases landed. The glossary, task types, and the
decision log now all run on the web substrate, and `sase memory web list` reports three
live webs in this project.

So the definitions are worse than merely thin — they actively mislead. An agent that
reads `glossary:memory-web` today is told the thing it is standing on does not exist
yet, and is told almost nothing about the shape it will actually meet on disk.

The current `Memory Web` body is also imprecise where it is not wrong:

- "A memory note that names a keyed collection of small sibling notes" never says the
  collection is a descriptor note plus a sibling _directory_, never names the
  `web: true` marker that makes a note a web, and never says how a strand is addressed.
- "A web renders as Core Memory or Reference Memory rather than being either render
  tier" is a distinction without a payload. The load-bearing fact it is groping toward
  is that the _descriptor body_ renders at the declared tier while _strand bodies never
  inline at all_ — which is the entire point of the substrate.

## Ground truth to encode

Verified against the tree and the CLI in this repo, not against bead notes:

1. A web is one flat descriptor note `sase/memory/<web>.md` carrying `web: true`, plus a
   sibling directory `sase/memory/<web>/<slug>.md` of strand files.
2. The descriptor carries `type: core | reference`; its own body renders at that tier.
   No strand body is ever inlined into `AGENTS.md` or a provider instruction shim.
3. The descriptor body carries a `<!-- sase:strands -->` roster block (`roster: inline`
   or `roster: list`, labelled by `roster_label`, nouned by `strand_noun`) that names
   every strand. The roster is how always-loaded context advertises what is available
   without paying for the bodies.
4. A strand's filename slug is its identity; its `keyword:` frontmatter (plus optional
   `aliases:`) is how it is addressed. Strands must not carry `type:` or `parent:`.
5. Reads go through `sase memory read <web>:<keyword>`; a bare `<web>` selector reads
   every strand.
6. `closure: mentions` (the glossary alone) expands a read into strands the requested
   strand's body mentions, recursively, capped by `-d N`. Every other web defaults to
   `closure: none`.
7. Strands scope-merge project-over-home per strand slug.
8. The three live webs here are `glossary` (core, 39 strands, `closure: mentions`),
   `decisions` (core, 6), and `task_types` (core, 5, generated from the bead registry).

Facts 6-8 are context for choosing wording; they must NOT all be crammed into a
three-line glossary strand. Facts 1-4 are what the term owes its reader.

## Scope: three strands, not one

The request named `Memory Web`. Two adjacent strands carry the identical false sentence
and cannot be left standing:

- `Memory Strand` is named inside any honest definition of `Memory Web`, and the
  glossary's `closure: mentions` therefore drags it into the same read. Fixing the
  parent while the child still says "Not yet implemented" would leave the lie in exactly
  the context the fix was meant to clean.
- `Strand Keyword` is the slug-vs-keyword distinction that `Memory Web` should _not_
  carry itself. It is where that fact belongs, so it has to be true for `Memory Web` to
  stay short.

All three are three-line strands. If the reviewer wants this narrowed to `Memory Web`
alone, say so on the approval gate; the implementer should otherwise do all three.

## Token-cost constraint

The reviewer's framing is that every token in context either helps or hurts. Two
consequences bind this work:

- **Length.** Glossary strands run roughly 40-70 words (`Stitch`, `Artifact`, `Chop`,
  `Agent Hood` are the calibration set). The new `Memory Web` body must land in that
  band. It is a dense term, so it may sit at the top of it, but it may not exceed ~75
  words.
- **Closure.** In a `closure: mentions` web, every glossary term a strand names is a
  transitive token cost on every read of that strand. Naming `Core Memory` and
  `Reference Memory` pulls both bodies at depth 1, and `Core Memory`'s own body then
  pulls `Sase Agent` -> `Agent Family` / `Sase Shell` at depth 2-3. Measure the real
  cost before and after rather than guessing.

The tier fact is judged worth its depth-1 cost, because "which of these two tiers does
this render at" is the single least-guessable thing about a web. The depth 2-3 tail
comes from `core-memory.md`'s own wording, not from this term; do not chase it here.

## Proposed bodies

These are drafts to beat, not text to paste unexamined. Keep the frontmatter blocks
(`keyword:`, and `Memory Strand`/`Strand Keyword` have no aliases today) exactly as they
are; change bodies only.

`sase/memory/glossary/memory-web.md`:

> A memory web is a keyed note collection: one flat descriptor note
> (`sase/memory/<web>.md`, marked `web: true`) plus a sibling directory of Memory Strand
> files. The descriptor body renders as Core Memory or Reference Memory per its `type:`
> and carries a roster naming every strand; strand bodies never inline, and are read on
> demand with `sase memory read <web>:<keyword>`.

`sase/memory/glossary/memory-strand.md`:

> One note inside a Memory Web, stored at `sase/memory/<web>/<slug>.md`. A strand body
> is never inlined into a generated document; it is fetched on demand by Strand Keyword.
> A project strand overrides a home strand with the same slug.

`sase/memory/glossary/strand-keyword.md`:

> The `keyword:` a Memory Strand is addressed by, as `<web>:<keyword>`, together with
> any `aliases:`. It is distinct from the strand's filename slug, which is its identity.

## Steps

1. Read `sase/memory/glossary/memory-web.md`, `memory-strand.md`, and
   `strand-keyword.md`, plus `sase/memory/decisions/memory-webs.md` (the accepted
   decision record) and `sase/memory/glossary.md` (the web descriptor). Confirm facts
   1-8 above still hold; the substrate is young and may have moved.
2. Record the current closure cost as a baseline:
   `sase memory read glossary:memory-web -r "baseline closure cost before rewrite"` and
   note how many strands and lines come back.
3. Rewrite the three bodies. Match the surrounding house style: present tense,
   definitional, no "is used to" hedging, no marketing, backticked literals for on-disk
   names. Do not introduce a new glossary term, and do not name a term that is not
   already a strand.
4. Run `sase memory init`. The reviewer explicitly authorized this glossary edit in the
   conversation that produced this plan, and that authorization covers regenerating
   `AGENTS.md`, the provider instruction shims, and the memory README. Do not ask again,
   and do not treat this plan file as the authorization for any _other_ memory edit.
5. Re-measure with a fresh `sase memory read glossary:memory-web`, and read the two
   sibling strands to confirm they render. Compare the strand count and line count
   against the step-2 baseline, and report both numbers in the completion summary.
6. Run `just install` (the workspace may be cold), then `just check`.

## Acceptance criteria

- No file under `sase/memory/` contains the string "Not yet implemented".
- `Memory Web`'s body states all four of: descriptor-plus-strand-directory shape, the
  `web: true` marker, that the descriptor body renders at its declared tier while strand
  bodies never inline, and the `<web>:<keyword>` read form.
- `Memory Web`'s body is at most ~75 words; the other two stay at or below their current
  length.
- No new glossary term is introduced, and every capitalized term named in the three new
  bodies resolves to an existing strand.
- `sase memory read glossary:memory-web` succeeds and its mention closure is no deeper
  and no larger than the step-2 baseline.
- `sase memory init` was run; `sase validate` reports no memory drift.
- `just check` passes.

## Risks

- **Pre-existing `init memory --check` drift.** Epic `sase-sq` carries three notes
  (2026-08-24) about `just check` failing only at `init memory --check` over generated
  README/shim churn, and its land note claims a template fix. If `just check` fails only
  there and the diff is generated-file churn unrelated to these three bodies, say so
  explicitly in the completion summary rather than silently absorbing it or silently
  ignoring it.
- **Roster churn.** These edits change bodies only, not `keyword:` values, so the
  glossary roster line in `AGENTS.md` should not move. If it does, something else
  changed - stop and report rather than accepting the diff.
- **Scope creep into the substrate.** `core-memory.md`'s depth 2-3 mention tail is real
  but is not this task. Leave it; if it seems worth fixing, note it as a proposed
  follow-up instead of editing it.
