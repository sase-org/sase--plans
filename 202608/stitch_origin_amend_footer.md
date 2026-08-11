---
tier: tale
title: Preserve the SASE_TYPE footer across amend, then land epic sase-jo
goal:
  Amending a commit through any SASE path keeps its SASE_TYPE footer, so a tracked
  stitch never silently reclassifies as manual, and epic sase-jo is closed out with
  every phase verified and every proposed follow-up dispositioned.
size: medium
proposed_by: bbugyi200.athena.sase-jo.land
bead: sase-jo
create_time: 2026-08-11 10:39:28
status: done
---

- **BEAD:**
  [sase-jo](https://github.com/sase-org/sase--beads/blob/main/pages/sase-jo/README.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-jo.land](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-jo.land.md)
- **COMMITS:**
  - [33b8861](https://github.com/sase-org/sase/commit/33b886150b672b85f471d0e3d1a9e9de0385cb71)
    — fix(vcs): preserve SASE_TYPE footer across commit amend

# Preserve the `SASE_TYPE=` footer across amend, then land epic sase-jo

## Context

Epic `sase-jo` ("Stitch origin indicators on the Artifacts Stitches sub-tab") shipped a
commit-origin classifier that labels every timeline row `stitch`, `auto`, or `manual`.
The classification is a lookup on the commit message footer, and it rests on one
invariant that the epic established and documented:

> Every commit SASE creates carries a `SASE_TYPE=` footer tag. Tracked
> `sase stitch create` commits carry `SASE_TYPE=stitch`; every automatic commit carries
> `SASE_TYPE=<kind>`. A commit with no `SASE_TYPE=` was not created by SASE.

That invariant is currently false. Four live SASE code paths amend an existing commit by
replacing its **entire** message with untagged text, which silently reclassifies a
tracked stitch as `manual` — exactly the misinformation the epic set out to remove.

Before this epic, `TYPE` had no behavioral consumers (only display styling in
`src/sase/vcs_log/_tag_style.py`), so dropping it on amend was harmless. The epic gave
`TYPE` behavioral meaning and published the invariant in `docs/commit_workflows.md`, so
closing this hole is the epic's own remaining work rather than a new feature.

The epic plan explicitly called for this check — "Confirm that the amend paths in
`src/sase/vcs_provider/plugins/_git_sync_ops.py`, `_git_core_ops.py`, and
`_git_commit_dispatch.py` preserve an existing footer rather than replacing it — an
amend that drops `TYPE` would silently reclassify a commit as `manual`." Phase
`sase-jo.2` performed that audit but concluded the callers round-trip the tagged
message. They do not.

## Evidence

`vcs_amend` in `src/sase/vcs_provider/plugins/_git_core_ops.py` runs
`git commit --amend -m <note>`, which replaces the whole message. Reproduced directly:

```
# commit message before amend
feat: tracked work

Body here.

SASE_TYPE=stitch
SASE_AGENT=someagent

# after: git commit --amend -m "[rewind] (3)"
[rewind] (3)
```

`classify_commit_origin` returns `stitch` for the first message and `manual` for the
second, so the Stitches row flips from `✦ stitch` to `✎ manual`.

## Confirmed broken callers

None of these round-trip the existing commit message. `Stitch.note`
(`src/sase/ace/patch/models/stitches.py`) is the one-line note text from a Patch
`STITCHES:` entry (`(N) Note text`), with the body in a separate `body` field — it never
carries a `SASE_TYPE=` footer.

1. `src/sase/workflows/rewind/workflow.py` — builds a wholly synthetic
   `amend_msg = f"[rewind] ({selected_entry_num})"`. Reachable from ACE through
   `src/sase/ace/tui/actions/hints/_rewind.py`, so this is a live user-facing action.
2. `src/sase/workflows/accept/workflow.py` — amends with `entry.note`, or
   `f"{entry.note} - {msg}"` when a message is supplied.
3. `src/sase/ace/change_actions.py` — same `entry.note` / `entry.note - extra_msg`
   pattern when accepting a proposal.
4. `src/sase/ace/scheduler/workflows_runner/completer.py` — amends with `entry.note`.

## Verified NOT affected — do not change these

- `vcs_reword` (`src/sase/vcs_provider/plugins/_git_sync_ops.py`): the ACE reword flow
  seeds the editor with the full current description including the tag block (see
  `_add_prettier_ignore_before_tags` in `src/sase/ace/handlers/reword.py`), so the
  footer round-trips unless a human deletes it.
- `vcs_reword_add_tag` (same file): appends one `KEY=VALUE` line to the existing full
  message fetched with `git log --format=%B`, preserving any footer structurally.
- `_amend_bead_changes` (`src/sase/vcs_provider/plugins/_git_commit_dispatch.py`): uses
  `--no-edit`, so HEAD's message is untouched.

## Implementation

Preserve the footer at the single choke point rather than patching four callers — this
is also what the `sase-jo.2` `PROPOSED FOLLOW-UP` note recommended.

- Teach `vcs_amend` in `src/sase/vcs_provider/plugins/_git_core_ops.py` to read HEAD's
  current message (`git log --format=%B -n1 HEAD`, the pattern `vcs_reword_add_tag`
  already uses in `_git_sync_ops.py`), extract its SASE footer tags, and re-apply them
  to the caller-supplied message before running `git commit --amend`. Reuse the shared
  footer grammar — `parse_commit_footer` and `update_trailing_commit_tags`, already used
  by `src/sase/workflows/commit/runtime_tags.py` — instead of hand-rolling tag parsing
  or string concatenation.
- Preserve the tags that describe provenance, and keep the existing `_KEY_PRIORITY`
  display ordering in `src/sase/vcs_log/_tag_style.py` so `TYPE` stays first. If the
  caller's own message already carries a tag block, the caller's value for a given key
  should win over the inherited one; inherited keys the caller did not set are carried
  forward.
- Make the operation idempotent: amending twice must not duplicate or reorder the tag
  block.
- If HEAD has no footer, or the `git log` read fails, fall back to today's behavior
  (amend with the caller's message unchanged) rather than failing the amend — a rewind
  that cannot read HEAD must not become a hard error.
- The four callers listed above should then need no changes. Confirm that by reading
  them; do not refactor them opportunistically.

## Contract test

`tests/test_commit_type_tag_contract.py` allowlists
`vcs_provider/plugins/_git_core_ops.py:vcs_amend` with the justification "Reword/rewind/
accept callers are responsible for preserving an existing footer. See sase-jo.2 PROPOSED
FOLLOW-UP for callers that currently build a fresh message instead of preserving one."
That justification no longer holds once `vcs_amend` preserves the footer itself.

- If the new implementation calls one of the helpers in `_TAG_HELPER_CALL_NAMES`, the
  file becomes self-evidently tagged and the allowlist entry must be removed —
  `test_allowlist_entries_are_all_still_present` will otherwise fail on the stale entry.
- If it does not, rewrite the entry's comment to state the real reason (`vcs_amend`
  inherits HEAD's footer) instead of pointing at callers.

## Tests

- Amending a commit that carries `SASE_TYPE=stitch` with a fresh untagged message keeps
  `SASE_TYPE=stitch` in the resulting message.
- A regression test using the literal rewind message shape (`[rewind] (N)`) asserting
  the origin classification stays `stitch` end to end, not just that a substring
  survives.
- Amending a commit with no footer leaves the caller's message untouched.
- Amending is idempotent across two consecutive amends.
- A caller-supplied `TYPE` wins over the inherited one.
- Cover at least one of the `entry.note` callers (accept or proposal-completion) so the
  fix is pinned at the workflow level, not only at the provider level.

## Verification

Run `just install` first — workspace directories are ephemeral and may have drifted
dependencies. Then run `just check-full`.

Two gate notes, both established while verifying this epic on 2026-08-11, so do not
re-derive them:

- `just check-full` excludes the PNG snapshot suite. It is already green for this epic
  (655 passed) and this change touches no rendering, so `just test-visual` is only
  needed if you change a renderer.
- The `test-cost` stage's `collection_cpu_seconds` budget is sensitive to other agents
  running pytest concurrently. Eight recordings on 2026-08-11 split 15–17s per worker
  (passing, budget 20s) against two ~47s spikes with an essentially flat node count. If
  it fails, check `uptime` and `pgrep -af pytest`, then re-run once the machine is
  quiet. Do not raise the limit in `tests/perf/baselines/test_cost_budgets.json` — that
  file's own notes forbid raising a limit to hide a one-off regression.

## Landing epic sase-jo

This is the final step of the plan; the agent that implements the fix also finishes the
landing. Do all of it after `just check-full` is green and the fix is committed.

### 1. File the one genuine follow-up

Use `/sase_new_task` for this, identifying epic `sase-jo` as the proposing bead:

- The `test-cost` `collection_cpu_seconds` gate fails spuriously when another SASE agent
  runs pytest concurrently. On 2026-08-11 two of eight recordings reported ~47s per
  worker against a 20s budget while six reported 15–17s, with node count flat at ~28,959
  — so the signal is machine contention, not suite growth. A concurrent `pytest` run in
  another workspace was confirmed active during one spike. The gate should either detect
  contention or measure collection cost in a way that is robust to it; raising the
  budget is explicitly the wrong fix. Not caused by this epic.

### 2. Close the epic

Close with `sase bead close sase-jo --note "<note>"`. The note should record that all
six phases were verified against the source and the epic's commits (`2d40e9297`,
`050264c7c`, `29af892b8`, `e1b39c72c`, `295f4e994` in this repo; `dc836c4` and `b6a1493`
in `sase-core`), that the amend-footer hole found during landing was fixed by this plan,
and the disposition of every `PROPOSED FOLLOW-UP` note:

- `sase-jo.2` (`vcs_amend` drops the footer) — **fixed by this plan**, and it was worse
  than the note described: the note assumed callers round-trip the tagged message, but
  all four build fresh messages.
- `sase-jo.4` (Ruff drift in `tests/test_external_mirror_issues.py`) — **declined,
  already resolved** by commit `c388b560c`; `ruff format --check` on that file is clean.
- `sase-jo.5` (2-value `manual`/`sase` core taxonomy vs the 3-value design) —
  **declined, resolved during the epic**. `sase-core` commit `b6a1493` replaced the
  2-value enum with `Stitch`/`Auto`/`Manual`, and `295f4e994` aligned the Python filter,
  parser, help text, schema, and docs. Verified live: the PyO3 binding returns
  `stitch`/`auto`/`manual` for all four precedence rules, and Python and Rust both
  report VCS-log wire schema 4.
- `sase-jo.6` (bump the declared `sase-core-rs` floor) — **declined by design**. A
  checkout ahead of the published window is the normal dev state;
  `tools/validate_sase_core_rs_version._remediation` says "No action is needed: editable
  installs build from the checkout regardless, and `tools/ratchet_core_window` moves the
  published window on the release branch at release time." `check-full` runs that probe
  with `--advisory` for the same reason.
- The epic bead's own two `DISCOVERED ISSUE` notes (the `vcs_log_wire_schema_version`
  3→4 pin blocking `just fix` repo-wide) — **resolved** by `2d40e9297`, which moved
  `tools/validate_sase_core_rs` to 4. `just install` and the `SASE validation` gate both
  pass.

If the close is rejected, the named phases were never completed: finish or reopen them,
or record the outcome deliberately with
`--force --reason ... --resolution canceled|superseded`. Never force merely to make the
command succeed.

### 3. After closing

- Run `just symvision`. Epic-symbol whitelist entries expire at close, so remove any
  stale entries and unused code it reports. Note that the `Justfile` currently has no
  `--epic-symbol` entries for `sase-jo` (the only two, for `sase-jd.5`, were removed by
  commit `e1c3d477b`), so this is expected to be a no-op — but run it and confirm.
- Set `status: done` in the frontmatter of the epic's plan file,
  `plans:202608/stitch_origin_badges.md` (resolve the concrete path with
  `sase bead show sase-jo`). It is currently `status: wip`.

## Out of scope

- Do not change the origin taxonomy, glyphs, legend, filter grammar, or docs. Those are
  verified working end to end.
- Do not backfill or rewrite existing commit messages. Classifier rule 3 (`AGENT`,
  `BEAD`, or `PLAN` present with no `TYPE` implies `stitch`) already covers history.
- Do not touch the Patch `origin:` PR-origin property (values `sase`, `external`,
  `unknown`). It is a separate grammar over a separate entity and the epic deliberately
  left it alone.
- Do not hand-edit the `sase-core-rs` requirement window in `pyproject.toml`.
