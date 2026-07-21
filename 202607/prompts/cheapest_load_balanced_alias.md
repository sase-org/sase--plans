---
plan: 202607/cheapest_load_balanced_alias.md
---
 %xprompts_enabled:false
# Previous Conversations

You are forking from 1 prior source. Source sections are independent parents, and section order carries no priority. Members inside an agent family section are sequential: each member continued the previous member's work. Carry forward relevant goals, constraints, decisions, and unfinished work with attribution when it matters. The New Query is the active request and takes precedence over conflicting source instructions.

## Source 1 of 1 — agent family `gy`

- **Members shown:** 2 of 2 (sequential chain, oldest first)

Family members ran as one sequential chain: each member continued the previous member's work, and the last member reflects the family's final state. These are transcripts of prior agents' conversations, not your own — attribute decisions to the named member when it matters.

### Member 1 of 2 — agent `gy--plan`

- **Outcome:** `completed` · **Model:** `codex/gpt-5.6-sol` · **Launch:** `20260721082044`
- **Transcript:** `~/.sase/chats/202607/gh_sase_org__sase-ace_run-gy__plan-260721_082044.md`

**User:**

Can you help me split the `@phase_worker` model alias up into a new `phase_worker` model alias bucket?

- Let's continue to define the `@phase_worker` model alias as is, but define it in the new `phase_worker` bucket.
- Let's also start defining the new `@small_phase_worker`, `@medium_phase_worker`, and `@large_phase_worker` model aliases.
- All of these should default to using the `@phase_worker` model alias as their value.
- We should use these new model aliases for the phase agents that we launch (which alias we use should depend on the size of the phase).

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.

**Assistant:**

*Plan submitted for review.*

**Plan file:** `/home/bryan/.sase/plans/202607/phase_worker_alias_bucket.md`

> # Plan: Size-specific phase-worker model aliases
> ## Context and outcome
> SASE currently exposes `@phase_worker` for small and medium phase agents and selects `@smartest` directly for large
> phase agents. The Models panel displays `phase_worker` as an individual role row. Introduce `@small_phase_worker`,
> `@medium_phase_worker`, and `@large_phase_worker` as first-class implicit builtin aliases, use them for their matching
> phase sizes, and group them with the existing `@phase_worker` alias in an always-present `phase_worker` bucket.
> Keep `@phase_worker` itself unchanged: when it is not configured, it falls back to `@default`. Each new size alias must
> fall back to `@phase_worker`, so existing persistent, launch-scoped, and temporary `phase_worker` overrides continue to
> govern all phase sizes unless a size-specific alias is overridden. Preserve `@smartest` as a resolvable/configurable
> alias for compatibility and explicit use, but stop selecting it automatically for large phases; users who want that

*See full plan file for details.*

### Member 2 of 2 — agent `gy--code`

- **Outcome:** `completed` · **Model:** `codex/gpt-5.6-sol` · **Launch:** `20260721083330`
- **Transcript:** `~/.sase/chats/202607/gh_sase_org__sase-ace_run-260721_082044.md`

**User:**

@sase/repos/plans/202607/phase_worker_alias_bucket.md

The above plan has been reviewed and approved. Implement it now.

**Assistant:**

I’ll implement the approved plan in the current SASE checkout, first checking the plan, repository state, and any scoped instructions. I’ll then make the changes and run the required install/check verification.

The plan touches generated SASE skills and the linked chezmoi source, so I’m applying the required `sase_memory_read` workflow first, then `sase_repo` before interacting with that linked repository. This will determine the exact regeneration order and which generated files must remain untouched by hand.

The required workflow confirms that only `src/sase/xprompts/skills/` is edited directly; installed skill files are regenerated with `sase skill init --force` and deployed with `chezmoi apply`. The checkout is clean, and I’m now tracing the centralized alias policy, Models view, phase selector, schema/completion surfaces, and their tests before patching.

Because the implementation changes Models-panel navigation and rendering, the repository instructions also require the `tui_perf` memory. I’m consulting that before touching the TUI layer so the new bucket does not regress refresh or selection behavior.

The implementation shape is clear: add three implicit role aliases with `@phase_worker` fallback, generalize built-in bucket aggregation for both `coders` and `phase_worker`, and make the shared selector map normalized sizes directly to those aliases. I’m installing the workspace dependencies now, then I’ll patch the policy and focused tests before updating docs and generated artifacts.

The core policy, selector, bucket aggregation, completion/schema surfaces, focused tests, and user docs are now updated. The generic fallback remains intact, while launches emit only size-specific aliases; I’m running the focused alias, Models-panel, bead-rendering, schema, doctor, and skill-source tests now to catch integration mismatches before regenerating snapshots and installed skills.

The focused test command hit the known local Rust-core version-window mismatch during dependency re-resolution, even though `just install` successfully built and installed the linked core. I’m using the already-installed workspace virtualenv directly for focused iteration; the required `just check` remains the final gate.

Focused coverage is green after the navigation adjustment. The generated-skills workflow now requires its materialization step: I’m running `sase skill init --force` from the edited canonical source, then `chezmoi apply` to deploy those generated provider skill files as prescribed; I’ll inspect both the main and linked-repo diffs afterward.

The regeneration command completed but reported all 85 outputs unchanged, while the linked chezmoi templates still contain the old routing text. I’m tracing the initializer’s source-selection rules before applying anything so the deployment uses this checkout’s edited canonical source rather than a stale installed resource.

`chezmoi apply` succeeded, but the live provider skills still show the old sentence. The initializer’s “unchanged” result is therefore inconsistent with its canonical source. I’m checking the rendered target objects and chezmoi’s own source/diff view to identify whether path layout or deferred deployment is masking the update, then I’ll rerun the supported workflow once corrected.

The inconsistency was executable scope: bare `sase` resolved outside this workspace, while `.venv/bin/sase` correctly regenerated five provider outputs and `chezmoi apply` deployed them. I’m also regenerating those same five files into the audited linked chezmoi checkout returned by `sase repo open`, using the initializer’s rendering API rather than editing generated `SKILL.md` files by hand, so the linked-repo change is tracked with this implementation.

The canonical and all five generated provider skills are now synchronized, and the live files were deployed successfully. I’m refreshing only the Models-panel visual goldens affected by the new bucket/name-column layout, including the new drilled-in `phase_worker` state, then I’ll rerun the visual suite without update mode.

The targeted visual update changed only Models-panel goldens, as expected; the wider alias-name column also affects existing coder/custom-bucket frames. I’m visually inspecting the collapsed and drilled-in phase states to confirm the long size aliases remain readable and the state/model columns still fit before accepting those goldens.

The updated snapshots are visually sound and pass exact comparison. I’m at the repository-wide gate now: `just check` will validate formatting, lint/type analysis, SASE contracts, committed plans, the full pytest suite, and all visual snapshots together.

The repository-wide gate reached mypy and found one local typing issue in the bucket-member sort callback; all earlier formatting and Ruff stages passed. I’m replacing that callback with a typed partial, then rerunning the full gate from the start.

`just check` now passes in full: formatting, all lint/type checks, SASE validation, committed-plan validation, and the complete test suite including visual snapshots. I’m doing the final diff audit now, including the five generated chezmoi skill files, to ensure there are no unrelated edits or stale old-routing references.

The linked checkout’s generated-skill audit reports all 85 targets current with zero stale files. The second full `just check` has again passed every static/validation stage and is finishing the complete test suite.

Implemented the approved plan.

- Added `@small_phase_worker`, `@medium_phase_worker`, and `@large_phase_worker`, each falling back through `@phase_worker` in [config.py](/home/bryan/.local/state/sase/workspaces/sase-org/sase/sase_16/src/sase/llm_provider/config.py:67).
- Routed phases by normalized size while preserving explicit-model precedence and `#plan` behavior in [work.py](/home/bryan/.local/state/sase/workspaces/sase-org/sase/sase_16/src/sase/bead/work.py:286).
- Added the built-in `phase_worker` Models-panel bucket, custom coalescing, metadata handling, and full-name rendering in [alias_view.py](/home/bryan/.local/state/sase/workspaces/sase-org/sase/sase_16/src/sase/llm_provider/alias_view.py:73).
- Updated completions, schema, doctor diagnostics, configuration examples, documentation, tests, and PNG snapshots.
- Regenerated all five provider `sase_plan` skills and successfully ran `chezmoi apply`. Audit: 85 generated targets checked, 0 stale.
- `@smartest` remains configurable for explicit use but is no longer selected automatically.

Verification passed:

- `just check`
- Models-panel visual snapshots: 11 passed with exact comparison
- Full generated-skill consistency audit

No commits were created.

The listed changes are the implementation changes from this session, so I’m applying the required `sase_git_commit` finalizer workflow to both the main checkout and the linked chezmoi checkout, then I’ll verify each repository is clean.

The diff audit matches the approved implementation: 42 tracked main-repo files plus one new Models-panel golden, and exactly five regenerated chezmoi skill files. The main branch is two commits behind upstream, which the commit wrapper is designed to reconcile after creating the local commit.

The main change is a user-facing routing change, and large phases no longer implicitly select `@smartest`; I’m marking that compatibility impact explicitly in the main conventional commit. The chezmoi commit is documentation-only generated output and will be committed separately.

The main commit completed, rebased cleanly over the two upstream commits, and is synchronized with `origin/master` at `eb93169eb`. I’m committing the five regenerated provider skill files now, then I’ll perform the required clean-status check in the linked repository.

The chezmoi commit itself was created, but its post-commit `chezmoi update` hook failed because that hook detected an unstaged change. Per the commit workflow I’m not creating another commit; I’m inspecting the exact residual state, then I’ll use the required resume path once it is safely resolved.

The linked checkout itself is clean and synchronized; the repeated failure is coming from chezmoi’s separate live source checkout, not from the repository I just committed. To resolve that post-hook without bypassing repository safeguards, I’m reapplying the required `sase_repo` workflow before inspecting that external checkout.

The live source has exactly the same five generated-file edits and no unrelated changes. I’m confirming the patch is byte-for-byte identical to the pushed chezmoi commit; if so, I’ll move that duplicate patch into a recoverable stash, resume the post-hook so it can pull the committed version, and then verify the live source and audited checkout both match upstream.

The resumed post-hook succeeded. Both the audited chezmoi checkout and live source are now clean, synchronized, and at `344fa4bd`. I’m removing only the temporary duplicate stash I created for recovery—the identical content is safely preserved in that pushed commit—then doing one final status check across both repositories.

Implemented, committed, and pushed:

- Main SASE repo: `eb93169eb feat!: route phase workers through size aliases`
- Chezmoi repo: `344fa4bd docs: update generated phase routing skill guidance`

Both repositories are clean and synchronized with `origin/master`. The chezmoi post-commit hook completed successfully after reconciling duplicate live generated files. `just check` passed.

---

%xprompts_enabled:true
# New Query

 Can you now help me add a new `@cheapest` builtin model alias
as well that is configured to load-balance between the `claude/opus@medium` and
`codex/gpt-5.5` models?

- We will need to add support for model alias load-balancing for this new model
  alias, which should have a value of `claude/opus@medium | codex/gpt-5.5` (we will
  use `|` to specify that a model alias uses load balancing).
- When a sase agent is launched that configures a load-balancing model alias
  (e.g. using the `%model` directive), we should select the next model in the
  load-balancers model pool for this alias (i.e. the list of models that were
  provided, using `|` to separate them) to use when launching this agent. This
  means we will need to start tracking when these model aliases are used.
- We should also be able to use this model alias load-balancing syntax to ensure
  a fallback is used in-case one of the specified model's agent CLI providers is
  not installed (ex: if the user does not have codex installed, the `@cheapest`
  model alias should always render to `claude/opus@medium`).
- We should configure the `@small_phase_worker` model alias to use `@cheapest`
  as its value. Also, let's start defining `@large_phase_worker` to use
  `@smartest` as its value.

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 