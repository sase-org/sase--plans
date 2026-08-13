---
tier: tale
title: Integrate canonical prompt search with stored rendered prompts and land sase-e7
goal:
  Canonical prompt search matches archived prompts on the text their author wrote, never on the rendered agent prompt
  the archive now stores beside it, the documented cross-store collapse fires again, and epic sase-e7 is closed on
  verified evidence with every proposed follow-up dispositioned.
size: medium
proposed_by: bbugyi200.athena.sase-e7.land
bead: sase-e7
create_time: 2026-08-02 12:17:49
status: wip
---

- **PROMPT:**
  [prompts/202608/canonical_search_body_integration.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/canonical_search_body_integration.md)
- **PARENT:**
  [202608/finish_dh_canonical_archive.md](https://github.com/sase-org/sase--plans/blob/main/202608/finish_dh_canonical_archive.md)
- **BEAD:** [sase-e7](https://github.com/sase-org/sase--beads/blob/main/pages/sase-e7/README.md)

# Integrate canonical prompt search with stored rendered prompts, then land `sase-e7`

## Why this plan exists

Epic `sase-e7` made the agents sidecar the canonical and only home for prompt Markdown. Its Phase 2 retargeted
`sase prompt search` at that archive (`feat(prompt)!: use the canonical prompt archive`, `53b1fc037`) and its Phase 4
rewrote `docs/prompt.md` to describe the new behavior (`docs: update prompt archive docs and plans map`, `af0a6b818`).

Roughly an hour after those landed, a phase of the still-active epic `sase-e6` landed
`feat(prompt-archive): store rendered prompts and link xprompts` (`f578c0aa4`). It appends the **final model prompt**,
verbatim, to every newly published archive entry as a sentinel-delimited collapsed section. That store-format change
could not integrate with `sase-e7`'s search work while `sase-e7` was still open, and the two now conflict: the store
`sase prompt search` reads is no longer only the prompt a human wrote.

`sase-e6` Phase 6 (`Read surfaces, docs, and end-to-end verification`, still in progress) enumerates the read surfaces
it will teach about the new sections — the ACE chat detail widget, `sase agent prompts show`, `sase chat show`, and
`docs/ace.md` / `docs/xprompt.md` / `docs/agent_images.md`. `sase prompt search` is not on that list, because that
surface did not read the archive when the `sase-e6` plan was written. Closing this gap is `sase-e7` integration work.

Both gaps below were reproduced against current `master`, not inferred.

### Gap 1 — canonical search indexes the machine-rendered prompt

`_load_archive_file()` in `src/sase/prompt/search/sources.py` builds a hit's searchable text with
`text = parse_plan_header_block(content).body.strip()`, which is the whole document body. Since `f578c0aa4`,
`render_prompt_document()` in `src/sase/agents_sync/prompt_archive/render.py` ends every document with
`render_prompt_sections(None, rendered_prompt)`, so that body now also carries the rendered agent prompt inside a
`<!-- sase:section:rendered -->` … `<!-- /sase:section:rendered -->` block.

Reproduced with an archive-shaped document whose authored prompt is 48 characters:

```
TITLE: Fix the tasks-tab selection bug in the ACE TUI.
TEXT LEN: 539 | user prompt len: 48
body-matches 'symvision': True      # word appears only in the rendered section
body-matches 'kubernetes': True     # word appears only in the rendered section
```

The indexed text also carries the raw `<!-- sase:section:rendered -->`, `<details>`, `<summary>`, and fence markup, so
snippets and previews render machine scaffolding.

This is not a small amount of foreign text. Rendered agent prompts written today under the per-project artifacts store
run from about 120 bytes to just over 13 KB, and the large ones embed prior-conversation transcripts: one 13,256-byte
rendered prompt begins `# Previous Conversations`, quotes another agent family's transcript, and inlines a different
plan's body. Every prompt archived from now on makes another agent's words searchable as if the user had written them,
so precision degrades monotonically as the archive grows.

No entry in the published archive carries a rendered section yet (`grep -rl 'sase:section:rendered' prompts/` in the
agents sidecar returns nothing across 2,943 files), which is exactly why this must be fixed before the archive fills
with them.

`src/sase/history/chat_prompt_sections.py` already exports the seam this needs: `strip_prompt_sections()` removes
complete sentinel-delimited sections. It currently has **zero** callers and is carried in the `Justfile` symvision
recipe as `--epic-symbol 'sase-e6(strip_prompt_sections)'`.

### Gap 2 — cross-store de-duplication can no longer fire

`docs/prompt.md` promises: "When `-s all` finds the same prompt in both stores (identical text), the local copy
collapses into the archive hit, annotated `also in local history`." `_dedup_hits()` implements that by comparing
`text_sha256`.

Before `sase-e7`, the non-local store held `prompt export --sdd` snapshots, which wrote the history record's own
`sha256` into frontmatter, and `_load_sdd_file()` preferred that recorded digest. Canonical archive entries have **no
frontmatter at all**, so `_load_archive_file()` falls through to `_sha256(text)` over a body that has been
prettier-reflowed, has had its header block stripped, and has had artifact references rewritten into Markdown links.
Exact-byte equality with a raw history record is now structurally impossible.

Measured against the live stores:

```
archive hits: 2937 local hits: 3951
digest overlap (dedup collapses): 0
local records with whitespace-normalized archive twin: 23
of those, byte-identical after strip: 0
```

23 prompts genuinely present in both stores are shown twice, and `also_in_local` is never set. `sase-e7` caused this by
swapping the store, so `sase-e7` owns it.

There is a second, time-delayed cause that a whitespace-only fix would not survive. `sase-e6` also rewrites every
resolvable xprompt reference in an archived prompt into a Markdown hyperlink, so an author's `#sase_plan` is stored as
`[#sase_plan](https://…)`. Measured on the same live stores:

```
normalized twins: 23
  of which carry xprompt refs: 23
local records carrying xprompt refs: 3799 / 3951
archive entries with linkified xprompt refs: 0
```

Every one of the 23 collapsible pairs carries an xprompt reference, and 96% of local history does. No archive entry is
linkified yet only because the feature landed the same day. A comparison key that ignores linkification would therefore
fix the collapse today and then quietly stop working for almost every prompt as linkified entries accumulate — the same
shape of failure as Gap 1. Tag extraction is unaffected: `summarize_prompt_for_list()` recovers `#sase_plan` from both
the bare and the linked spelling (verified), so only the de-duplication key needs to account for this.

## Out of scope

- Do not change what the archive **writes**. The document format belongs to the in-flight `sase-e6` epic; this plan
  changes only how the search layer **reads** it.
- Do not add a search flag for querying rendered prompt text. That is a new feature, not integration; if it seems
  worthwhile, record it with `/sase_new_task` instead.
- Do not rewrite published archive entries.
- Do not edit anything under `sase/memory/` or any generated agent instruction file (`AGENTS.md`, `CLAUDE.md`,
  `GEMINI.md`, `OPENCODE.md`, `QWEN.md`). If one of them needs updating, record it with `/sase_new_task`.

## Step 1 — index only the authored prompt body

1. In `_load_archive_file()` (`src/sase/prompt/search/sources.py`), strip the sentinel-delimited machine sections from
   the parsed body before it becomes the hit's `text`, using `strip_prompt_sections()` from
   `src/sase/history/chat_prompt_sections.py`. Import it directly rather than reimplementing sentinel handling; a second
   copy of the sentinel constants is exactly the drift this seam exists to prevent.
2. Keep the existing tolerance contract: a document that cannot be read or parsed is still skipped rather than failing
   the whole scan, and stripping must never raise on a malformed or half-written section. `strip_prompt_sections()`
   already leaves an unpaired sentinel alone; confirm that behavior and make sure a document with an unterminated
   rendered section still yields a usable hit rather than an exception.
3. Re-derive `title` and `tags` from the stripped text so a machine section can never supply either.

## Step 2 — restore the documented cross-store collapse

1. Change `_dedup_hits()` (`src/sase/prompt/search/sources.py`) to compare a normalized content key instead of raw
   `text_sha256`. Apply the same normalization to both sides so the rule stays symmetric:
   - collapse the linkified spelling of an xprompt reference back to the authored one — `[#name](<url>)` becomes
     `#name`. Without this the collapse dies for 96% of prompts as `sase-e6`'s linkification reaches the archive;
   - then normalize whitespace (`" ".join(text.split())` is sufficient and is what the measurements above used).
     Prettier reflow is a formatting difference, not a content difference, and the archive is prettier-formatted by
     construction.
2. Leave `PromptHit.text_sha256` itself untouched — it is part of the `-f json` shape and of local-history identity.
   Only the comparison key changes.
3. Preserve the rest of the contract: archive hits stay ahead of local hits, a collapsed archive hit is annotated
   `also_in_local=True`, the collapsed local hit is dropped, and a hit that cannot be proven a duplicate is never
   collapsed.
4. After the change, re-run the measurement in Gap 2 against the live stores and record the new collapse count in the
   bead note. It should be 23 or more, not 0.

## Step 3 — tests

Add regression coverage to `tests/prompt_command/test_search_sources.py` (58 tests there pass today; keep them passing):

1. An archive entry carrying a rendered section indexes only the authored prompt: a term that appears **only** inside
   the rendered section does not match, the authored term does match, and `text` contains no `<!-- sase:section:`
   sentinel, `<details>`, or `<summary>` markup.
2. The same for a document carrying an XPrompt section, so the fix does not depend on which of the two sections the
   writer emitted.
3. A malformed document with an opening rendered sentinel and no closing sentinel still produces a hit and does not
   raise.
4. An archive entry whose authored text differs from a local-history record only by line wrapping collapses: the archive
   hit is returned with `also_in_local=True` and the local hit is absent.
5. The same collapse holds when the archive entry's xprompt references are linkified (`[#sase_plan](https://…)`) and the
   local record's are bare (`#sase_plan`) — the case that covers 96% of real prompts.
6. Two genuinely different prompts do not collapse.

Build the fixtures by calling `render_prompt_sections()` rather than hand-writing sentinels, so the tests track the
shared format.

## Step 4 — documentation

In `docs/prompt.md` (owned by this epic; `docs/ace.md`, `docs/xprompt.md`, and `docs/agent_images.md` belong to
`sase-e6` Phase 6 — leave them alone), state in the search section that archived entries are matched on the prompt as
authored, and that the stored rendered agent prompt is deliberately not part of the searchable text. Reconcile the
`also in local history` sentence with the normalized-text rule from Step 2 so the documented condition is the
implemented one.

## Step 5 — verify

1. `just install`, then `just check`.
2. If the full suite fails only in the known host-contention tests named in the follow-up disposition below, rerun those
   exact tests in isolation and record both results; do not treat a reproduced isolated failure as inherited.
3. Confirm `sase validate` still passes and that `sase agent prompts validate` reports no new errors.
4. Commit through the SASE commit workflow.

## Step 6 — land `sase-e7`

This step runs only after Steps 1–5 are committed and pushed.

1. File the two genuinely distinct, non-epic-caused follow-ups with `/sase_new_task`, naming the proposing beads and
   choosing an intentional `--size` for each:
   - **Source-free prompt-archive project disambiguation** (proposed by `sase-e7.1` and again by `sase-e7.5`): with
     published `sase-core-rs` 0.17.11 and all SASE launch environment removed, `sase agent prompts validate` exits with
     `multiple projects matched; pass -p/--project` (raised by `_resolve_target()` in `src/sase/agents/cli_prompts.py`)
     instead of selecting a project or requiring one at the top-level validation boundary.
   - **Host-contention test hardening** (proposed by `sase-e7.1`, `sase-e7.2`, `sase-e7.3`, `sase-e7.4`, and
     `sase-e7.5`): under saturated multi-worker runs these fail and then pass immediately in isolation — chiefly
     `test_concurrent_bead_mutations_wait_past_the_old_lock_timeout`
     (`tests/test_bead/test_cli_work_contention_regressions.py`, untouched by `186fd2010`, which hardened
     `tests/test_sdd_git_contention.py` instead), plus fs-watcher coalescing, artifact-files modal copy, and
     `test_bulk_waiting_agents_mount_forced_artifact_prompts` (`tests/ace/tui/test_agent_bulk_kill_edit.py`).
2. Record, without filing, the two follow-ups already resolved upstream while this epic ran — verify each claim before
   asserting it:
   - `test_scaled_suite_runs_share_capacity_and_release_after_sigkill` (proposed by `sase-e7.5`) was resolved by
     `abbeb36d9`, `test: make suite-gate integration budgets load-tolerant`.
   - the retry-countdown PNG snapshot flake (proposed by `sase-e7.3`) appears resolved by `adfa35043`,
     `test(visual): stabilize PNG convergence under contention`, which rewrote
     `tests/ace/tui/visual/test_ace_png_snapshots_agents_retry_e2e.py`. If it is not resolved, fold it into the
     host-contention task instead of dropping it.
3. Close the epic with `sase bead close sase-e7 --note "..."`. The note must cover: child verification against source
   (not bead text), the `sase-e6.5` conflict found and fixed here with the new collapse count from Step 2, the published
   `sase-core-rs` version, sidecar validation counts, the full-suite result, and every follow-up disposition including
   why each recorded item was not filed. Do not reach for `--force` merely to make the close succeed.
4. Only after the close succeeds, run `just symvision`. Note that no `sase-e7` epic-symbol whitelist entry exists in the
   `Justfile` today, so expect no `sase-e7` expiry; the only entry there is `sase-e6(strip_prompt_sections)`, and Step 1
   gives that symbol its first caller. Do **not** remove the `sase-e6` entry as part of this plan — that symbol belongs
   to an active epic and `sase-e6` Phase 6 has its own read surfaces to wire up. Remove only what Symvision actually
   reports as unused, then rerun the proportionate checks.
5. Set `status: done` in the frontmatter of `@plan:202608/finish_dh_canonical_archive.md`, leaving the rest of that plan
   intact, and publish the plans-sidecar edit through the normal plans workflow.
6. Finish by showing the closed epic and reporting clean, synchronized status for the main repo, the plans sidecar, and
   the agents sidecar.

## Verification

`just install` before `just check`, and `just check` before finishing. Use `/sase_repo` to reach the plans and agents
sidecars; never fetch their files over the web, and never name a numbered workspace directory in a commit, doc, or plan
edit.
