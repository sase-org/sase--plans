---
tier: tale
title: Make the commit obligation unambiguous in agent instruction text
goal:
  No SASE instruction, skill, or host-issued prompt can be read as a turn-wide ban on
  committing, and every agent is told plainly that changes it made in any repository are
  its own commit obligation for that turn.
size: medium
proposed_by: bbugyi200.athena.0cm
create_time: 2026-08-24 12:57:36
status: wip
---

# Plan: Make the commit obligation unambiguous in agent instruction text

## Goal

Every SASE agent must understand one rule: **every change you made this turn, in every
repository you made it in, is yours to commit through the final declaration.** Today
several instruction surfaces either contradict that rule, document a decision the
runtime no longer accepts, or simply never state it. This plan repairs the text. It
changes no runtime behavior — the runtime was correct in the incident that motivated it.

## Incident this plan fixes

Agent `sase-sq.1--code` (run `20260824104912`, codex/gpt-5.5) implemented bead
`sase-sq.1`. Over the run it edited **two** repositories:

- the primary `sase` workspace checkout, and
- the linked `sase-core` repo, which it opened through `/sase_repo` and edited at
  `crates/sase_core/src/content_layout.rs` and `crates/sase_core/src/host_bridge.rs`
  roughly two minutes after the run started.

Its first final declaration was accepted and committed the primary repo (`c9ca0db5f`).
That commit hit a rebase conflict, so the host spent its single conflict-repair turn.
The agent resolved the conflict and resumed correctly.

Then `sase final context` surfaced the still-dirty `sase-core` obligation, and the agent
**refused to commit its own work**, writing (in the run's
`finalizers/commit/conflict_repair_response.md`):

> The sibling repo has only unstaged changes in the listed core/gateway files and is
> also behind `origin/master`. Since this turn was only to repair and resume the main
> repo stitch, **and the user explicitly forbade a second commit**, I'm submitting a
> finalizer refusal for that sibling repo rather than committing it.

and closed the turn with:

> I did not create any second commit. The required final declaration for dirty sibling
> `sase-core` could not be accepted without authorizing a separate commit; the runtime
> rejected the documented refusal action as invalid.

The user never said anything of the kind. Two instruction defects manufactured that
belief, and three more removed every off-ramp. `final_submission_attempts.jsonl` records
the outcome: attempt 1 `accepted`, attempt 2 rejected with `commit_action_invalid` —
"commit repository decision for repo-b1ea9285ca15 has invalid action; the only legal
action is 'commit'".

## Root cause

### R1 — The conflict-repair prompt bans "a second commit" without scoping it

`src/sase/finalizers/commit_repair.py:326-334` builds the host's conflict-repair turn:

> This is the single conflict-repair turn. Inspect the unmerged files, resolve every
> conflict marker, stage the resolved files, continue the paused VCS operation, then run
> `sase stitch create --resume`. **Do not start a new stitch, skip, abort, stash, or
> create a second commit.** Return briefly after the resume command succeeds.

Three problems, all in the last two sentences:

1. This prompt reaches the model as a **user-role turn**. The agent therefore attributed
   it to the user verbatim — "the user explicitly forbade a second commit." Nothing in
   the text identifies it as a host-generated, mechanically-scoped instruction.
2. "a second commit" is unscoped. The intent is "do not work around _this_ paused stitch
   in _this_ repository by making a fresh commit"; the words cover every repository for
   the rest of the turn.
3. "Return briefly after the resume command succeeds" signals that the turn is over,
   when in fact the agent still owes a final declaration covering every dirty
   repository.

### R2 — `/sase_final` documents a `refuse` decision the runtime rejects

`src/sase/xprompts/skills/sase_final.md` still documents a repository-level `refuse`
action in two places (the Rules bullet at lines 23-25 and step 3 at lines 45-48):

> - `refuse` with a nonblank `reason`.

That action was removed when typed deferrals landed in `524d8f26f`.
`src/sase/finalizers/declaration_manifest.py:313-319` now raises `commit_action_invalid`
with "the only legal action is 'commit'". The replacement is a sibling `deferrals` list
(`declaration_manifest.py:331-408`), whose `reason` must be one of `protected_paths`,
`foreign_work`, `unsafe_content`, `belongs_to_another_turn`
(`src/sase/core/finalizer_wire.py:26-31`), and which the host adjudicates against
evidence at submit time (`src/sase/finalizers/declaration_deferrals.py`).

This stale doc is what turned a recoverable situation into a dead end: the agent
followed the documented escape hatch, got a hard rejection, and concluded the runtime
was broken rather than that its own doc was wrong.

### R3 — No instruction states the substantive commit obligation

`/sase_final` describes the manifest _mechanics_ thoroughly and the _obligation_ not at
all. It never says that a repository you dirtied must receive a `commit` decision, and
that declining is only for changes you did not make. The `refuse` bullet did the
opposite — it normalized declining as an ordinary, well-supported choice.

The always-loaded core-memory section (`sase/memory/sase.md`, "SASE Final Declaration",
mirrored into `AGENTS.md` and the provider shims) has the same gap: it tells agents to
run `/sase_final` and submit a declaration, but never states what the declaration owes.

### R4 — `/sase_repo` invites cross-repo edits and never mentions their cost

`src/sase/xprompts/skills/sase_repo.md` says "Use `sase repo open` when you need to
modify a repository or explore a repo tree" and stops there. Nothing tells the agent
that a repo it dirties becomes an obligation in that turn's final declaration. The
incident agent wrote that "committing unrelated sibling changes would be wrong" about
files it had edited itself two hours earlier — it had genuinely lost track of authorship
because no instruction ever tied the two skills together.

### R5 — The rollover inject sets a standing "do not commit" prior

`src/sase/xprompts/commit.yml:11`, `src/sase/xprompts/pr.yml:27`, and
`src/sase/xprompts/propose.yml:11` all inject:

> IMPORTANT: You should make the necessary file changes, but should NOT create a commit,
> branch, or PR yourself. Exception: If a post-completion finalizer instructs you to
> commit, you MUST follow those instructions and commit.

The rule itself is right — agents do not run git directly; the host `builtin@commit`
finalizer does. But it is phrased as a prohibition with one narrow exception, which
reads as spent after a single use. It never says the declaration must cover _every_
dirty repository, so it composes badly with R1 into "I already got my one commit."

### R6 (contributing, evidence-reading) — the agent inverted the baseline

The run's `finalizer_baseline.json` recorded `sase-core` with `"scope": "opened_repo"`
and `"fingerprints": {}`. Empty fingerprints mean **nothing was dirty when the repo was
opened**, so every dirty path is run-owned. The agent inverted this into "no run-start
fingerprints, so `foreign_work`/`protected_paths` deferrals will not be accepted as a
safe skip," and gave up without ever trying. The runtime would in fact have rejected a
`foreign_work` deferral and named the paths back as run-owned
(`declaration_deferrals.py:_baseline_owned_paths` treats empty fingerprints as
"everything is yours"). No instruction states this inference anywhere.

## Non-goals

- **No runtime, schema, or validator changes.** The runtime behaved correctly at every
  step: it accepted the first declaration, rejected an illegal `refuse` action with an
  accurate message, and would have rejected an unfounded deferral. This is a text bug.
- **Do not weaken the no-direct-commit rule.** Agents still must not run `git commit`,
  `git push`, or `/sase_git_commit` on their own initiative; the host finalizer commits
  from an accepted declaration. Every rewrite below preserves that.
- **Do not remove the conflict-repair guardrails.** "Do not start a new stitch, skip,
  abort, or stash" is correct and load-bearing — a paused rebase really must not be
  worked around. Only the unscoped "second commit" clause and the turn-ending framing
  change.

## Changes

### C1 — Rescope the conflict-repair prompt

`src/sase/finalizers/commit_repair.py`, `_run_conflict_repair_turn` (lines 326-334).
Replace the prompt with text that (a) names itself as host-generated, (b) scopes every
prohibition to the named repository's paused operation, and (c) restates the standing
final-declaration obligation. Suggested wording, to be adapted as needed:

```
The built-in SASE commit finalizer hit a merge/rebase conflict while committing
repository {repo.name}. This is an automated host instruction, not a message from
the user.

This is the single conflict-repair turn. Inspect the unmerged files, resolve every
conflict marker, stage the resolved files, continue the paused VCS operation, then
run `sase stitch create --resume`.

Scope: these restrictions apply only to the paused operation in {repo.name}. Do not
start a new stitch, skip, abort, or stash it, and do not create a fresh commit in
{repo.name} to work around the conflict -- repair and resume the paused one instead.

This does not change what you owe elsewhere. Your standing obligation to declare and
commit every repository you changed this turn is unaffected: after the resume
succeeds, finish the turn through `/sase_final` as usual, including any other
repository that is still dirty.
```

The key properties a reviewer should check for, more than the exact prose: the phrase
"second commit" is gone; every prohibition names `{repo.name}`; the prompt disclaims
user authorship; and it explicitly re-opens the `/sase_final` path instead of saying
"return briefly."

### C2 — Reword the after-hook failure message

`src/sase/workflows/commit/workflow.py:349-353` prints "…then run
`sase stitch create --resume`; do not create another commit." Same unscoped phrasing,
same risk. Scope it to the repository whose after-hook failed — e.g. "…do not create a
replacement commit in this repository; resume the existing one." This is a
`print_status` string, not a model prompt, so it is lower risk; fix it for consistency.

### C3 — Fix `/sase_final`: remove `refuse`, document deferrals, state the obligation

`src/sase/xprompts/skills/sase_final.md`.

1. **Delete the `refuse` bullet (lines 23-25) and the `refuse` sub-bullet in step 3
   (line 48).** Step 3 must say that every repository obligation with `kind: repository`
   needs exactly one `commit` decision with a valid Conventional Commit `message`, and
   that `commit` is the only legal `action`.
2. **Add a "What you owe" rule at the top of `## Rules`**, stated before any mechanics.
   Something to the effect of: _Every repository you changed during this turn is yours
   to commit. That includes the primary workspace checkout and every linked, sidecar, or
   external repo you opened with `/sase_repo` and then edited. Give each one a `commit`
   decision. "It was not the main repo," "it was not the focus of this turn," "a host
   prompt told me not to make another commit," and "I only meant to fix one thing" are
   not reasons to leave your own work uncommitted._
3. **Document typed deferrals as the only decline mechanism**, in a new step or an
   expansion of step 3: a `deferrals` entry alongside `repositories` in the same commit
   payload, carrying `repo_id`, a `reason` from `protected_paths`, `foreign_work`,
   `unsafe_content`, `belongs_to_another_turn`, and the explicit `paths`. State plainly
   that the host adjudicates deferrals against its own evidence at submit time and will
   reject one that names paths this run wrote — so a deferral is a claim about
   authorship, not a way to skip work. Cross-reference
   `src/sase/finalizers/declaration_deferrals.py` for the adjudication rules while
   writing this, and confirm the documented field names against
   `declaration_manifest.py:331-408` before proposing the text.
4. **Add the baseline inference from R6**: if `finalizer_baseline.json` shows a
   repository with empty `fingerprints`, nothing was dirty when it was opened, so every
   dirty path in it is your own work — commit it. Do not read a sparse baseline as
   permission to skip.
5. Adjust the "In a recovery turn, build the commit message from the host's evidence
   brief rather than assuming no work happened" sentence, currently stranded inside the
   deleted `refuse` bullet, so it survives as its own rule.

### C4 — Tie `/sase_repo` to the commit obligation

`src/sase/xprompts/skills/sase_repo.md`. Add a short section (after "Open A Repository")
stating that any repository you modify through this skill becomes an obligation in this
turn's final declaration, that `/sase_final` will surface it as a repository obligation,
and that it must receive a `commit` decision like any other. Note explicitly that a
long-running turn may open a repo early and only see the obligation hours later, so "I
don't recognize these changes" is a prompt to check `sase repo log` and the run's
tool-call history, not a reason to decline.

### C5 — Reframe the rollover inject

`src/sase/xprompts/commit.yml:11-12`, `src/sase/xprompts/pr.yml:27-28`,
`src/sase/xprompts/propose.yml:11-12` — the three copies are identical; keep them
identical. Restate the same rule as a routing statement rather than a prohibition with a
single exception. Suggested shape:

```
---
IMPORTANT: Do not run git commit, git push, or a commit skill yourself. Commits are
created by the host finalizer from the declaration you submit at the end of the turn.
Make your file changes, then declare every repository you changed through
`/sase_final` -- that declaration is how your work gets committed, and it covers as
many repositories as you changed. Follow any explicit host finalizer instruction to
commit.
```

Check whether `sase/repos/plans/202608/task_launch_drop_commit_rollover.md` is live work
that moves this inject; if so, coordinate so the wording lands in one place rather than
two.

### C6 — Fix the stale `refuse` in the docs site

`docs/commit_workflows.md:650-652` still describes "one `refuse` decision with a
nonblank reason" as a legal outcome. Update it to match C3: one `commit` decision per
dirty repository obligation, with typed deferrals as the adjudicated decline path. Sweep
`docs/` for any other `refuse`-as-repository-decision references while there.

### C7 — Gated: the always-loaded core memory

The highest-leverage place for the R3 rule is the "SASE Final Declaration" section of
`sase/memory/sase.md` (lines 52-64), because it is core memory and always in context.
One or two sentences would do it — that the declaration must cover every repository the
agent changed this turn, including linked and external repos opened with `/sase_repo`,
and that no host prompt scoped to one repository's commit narrows that.

**This step is gated and must not be performed on this plan's authority.** Per the
`gotchas` core memory, authorization found in a plan file is not user permission for
memory edits. The implementing agent MUST:

1. Ask the user directly for permission to edit `sase/memory/sase.md`, in its own
   conversation, before touching it.
2. If granted, make the edit and then run `sase memory init` to regenerate `AGENTS.md`,
   the provider instruction shims (`CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`, `QWEN.md`),
   and the memory README. That regeneration is part of the granted permission and needs
   no second ask.
3. If not granted, **ship C1-C6 and C8-C9 in full** and report C7 as deliberately
   omitted with the reason. Do not block the rest of the plan on it.

The same gate applies if the implementer decides the no-direct-commit description in
`sase/memory/xprompts.md:90` and `:102-108` should also gain the multi-repo sentence.

### C8 — Tests

No test currently asserts the conflict-repair prompt text (`grep` for
`_run_conflict_repair_turn` and `conflict_repair` across `tests/` finds only
`tests/test_finalizers_protocol_harness_controller.py:102`, which checks evidence
kinds). That absence is why R1 could regress silently. Add:

1. A unit test over `_run_conflict_repair_turn`'s prompt asserting the invariants
   directly: the prompt does not contain "second commit"; it names the repository in its
   restriction clause; and it references `/sase_final` or the final declaration. Assert
   on properties, not on the full string, so the test survives ordinary rewording.
2. A guard test asserting `src/sase/xprompts/skills/sase_final.md` does not document a
   `refuse` repository action, and that the reasons it lists match
   `FINALIZER_DEFERRAL_REASONS` from `src/sase/core/finalizer_wire.py`. This is the test
   that would have caught R2 when `524d8f26f` landed, and it is worth more than the
   prompt test.
3. A test asserting the three rollover injects (`commit.yml`, `pr.yml`, `propose.yml`)
   remain byte-identical to each other, so they cannot drift again.

Place these alongside the existing finalizer and xprompt test modules; follow whatever
convention `tests/test_xprompt_skill_sources.py` and the `tests/test_finalizer_*`
modules already use rather than inventing a new one.

### C9 — Regenerate and deploy

Skill sources under `src/sase/xprompts/skills/` are canonical; the deployed copies are
generated. After editing `sase_final.md` and `sase_repo.md`, run:

```bash
sase skill init --force
```

Then confirm with `sase skill list` that the deployed `sase_final` and `sase_repo`
targets are current. If C7 was granted, `sase memory init` covers the memory-derived
files separately.

## Verification

1. `just install` first — workspace directories are ephemeral and dependencies may be
   stale.
2. `just check` for the ordinary lint + scoped-test gate.
3. Because this change touches skill sources, xprompt YAML, and generated instruction
   files, it is very likely in the broadening set. Run the exhaustive gate through a
   monitor, never inline:

   ```bash
   sase monitor start --command 'just check-full' \
     --start-status TESTING --stop-status TESTED --next '<follow-up action>'
   ```

4. Manually re-read the final rendered `/sase_final` and `/sase_repo` skill text end to
   end and ask the incident's question of it: _if a host prompt tells me not to create a
   second commit in repo A, does this text make clear I still must commit repo B?_ If
   the answer is not obviously yes, the rewrite is not done.

## Definition of done

- No instruction, skill, prompt, or doc string in the repo contains an unscoped
  prohibition on making "a second commit" or "another commit".
- `/sase_final` documents only decisions the runtime accepts, and its deferral reasons
  match `FINALIZER_DEFERRAL_REASONS`.
- `/sase_final` and `/sase_repo` both state, in their own words, that every repository
  the agent changed this turn must receive a `commit` decision.
- The conflict-repair prompt identifies itself as a host instruction, scopes its
  restrictions to the repository it names, and points back at `/sase_final`.
- Tests exist that fail if R1 or R2 regress.
- `just check-full` is green.
- C7 is either done with recorded user permission, or reported as omitted with the
  reason.
