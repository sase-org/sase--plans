---
tier: tale
title: Retire the --q asker suffix and the question-asker role taxonomy
goal:
  "PLAN_CHAIN_QUESTION_SUFFIX, the q family role, and the root/phase-question suffix
  kinds are gone from sase.plan_chain and every consumer; an agent that asks a question
  is an ordinary agent shell, the question gate shell owns the QUESTION/ANSWERED status,
  and historical --q artifact dirs still canonicalize and render. "
size: medium
proposed_by: bbugyi200.athena.sase-ud.12
bead: sase-ud.12
create_time: 2026-08-27 01:49:05
status: wip
---

- **PARENT:** [202608/gate_shells.md](gate_shells.md)
- **BEAD:**
  [sase-ud.12](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ud/sase-ud.12.md)

# Goal

Delete `PLAN_CHAIN_QUESTION_SUFFIX` and the root/phase-question suffix taxonomy from
`sase.plan_chain` and every consumer, so that an agent that asks a question is just an
ordinary agent shell. The question gate shell (`<family>--gate-N`) already owns the
question, its `QUESTION`/`ANSWERED` statuses, and its follow-up launch; the `--q`
suffix, the `q` family role, and the `root_question`/`phase_question` suffix kinds are
what is left over from naming the _asking_ agent, and they are now vestigial.

This is the `q-suffix-cleanup` phase of the `gate_shells` epic (bead `sase-ud.12`). It
is a large but mechanical diff. It was pulled into the epic rather than filed as
follow-up because leaving it keeps a dead concept load-bearing in `plan_chain.py`, the
family roster, and the TUI status machinery that the next phase (`status-collapse`) has
to reason about.

# Background — what exists today

`src/sase/plan_chain.py` classifies a family member's suffix into a
`_PlanChainSuffixInfo` record with a `kind`. Three of those kinds exist only to name a
question asker:

- `legacy_question` — the literal `--q` suffix (`PLAN_CHAIN_QUESTION_SUFFIX`), plus its
  `.q` and `-q` entries in `_LEGACY_DOTTED_SUFFIX_MAP` / `_LEGACY_DASH_SUFFIX_MAP`.
- `root_question` — a bare numeric root token (`--0`, `--1`) or any root token whose
  stored `agent_family_role` is `q`. `--0` and `--1` are inferred as role `q` purely
  from the token value.
- `phase_question` — a phase suffix with a numeric tail (`--code-0`, `--plan-0-0`),
  meaning "the continuation of the `--code` phase after it asked a question".

Verified current classification (`.venv/bin/python`, `_parse_plan_chain_suffix`):

| suffix              | canonical    | role       | kind                  |
| ------------------- | ------------ | ---------- | --------------------- |
| `--q` / `.q` / `-q` | `--q`        | `q`        | `legacy_question`     |
| `--0` / `--1`       | same         | `q`        | `root_question`       |
| `--2`               | `--2`        | `feedback` | `legacy_feedback`     |
| `--code-0`          | `--code-0`   | `code`     | `phase_question`      |
| `--plan-0-0`        | `--plan-0-0` | `feedback` | `phase_question`      |
| `--reviewer`        | `--reviewer` | `None`     | `None` (unclassified) |

`--reviewer` is the shape an ordinary, non-plan-chain family member already has: it
canonicalizes, it is a family member, and it carries no inferred role. That is exactly
the shape `--q`, `--0`, and `--1` should have after this change.

Consumers of the taxonomy, all verified by grep:

- `axe/run_agent_exec_questions.py` — the flag-Off branch and the `%auto` short-circuit
  branch both call `is_root_question_suffix` / `question_followup_suffix_template` and
  pass `agent_family_role="q"`.
- `main/pipe_handler.py` — reserves `q` as a `sase pipe --name` token.
- `ace/tui/agent_context_members.py` and
  `ace/tui/widgets/prompt_panel/_agent_display_content.py` — `--q` suffix labels and
  `role == "q"` branches.
- `ace/tui/models/_agent_status_family_policy.py` — `has_inherited_family_question`,
  `is_answered_continuation_asker`, `is_answered_root_asker_step`, all gated on
  `agent_family_role(agent) == "q"`, re-exported through `_agent_status_family.py` and
  `_agent_status_overrides.py` and applied in `_agent_status_apply.py`.
- `ace/tui/models/_agent_status_family_core.py` — an explanatory comment.

The Rust core carries none of this: `crates/sase_core` has no `q` role, no `--q`
literal, and no suffix-kind taxonomy (checked via the `sase-core` linked repo). No
cross-repo change is needed.

The `gate_shell_handoff` beta flag is still live, so the Off branch of
`handle_questions_marker` must keep working after this change; `status-collapse` deletes
it in the next phase.

# Design

## 1. A root-token suffix becomes ordinary, not a question

Drop the `token in {"0", "1"}` role-`q` inference. A bare root token with no stored role
and no legacy feedback round parses as `None` — the `--reviewer` shape.
`canonical_plan_chain_suffix` still returns `--0`, `--1`, and `--q` unchanged, because
`_ROOT_TOKEN_SUFFIX_RE` already matches them. This is the read-side compatibility the
phase brief asks for: historical `--q` artifact dirs still split into a family base plus
a suffix, still render, and still label as `q` through the generic suffix-token fallback
— but `--q` is no longer a live suffix any code can allocate.

The `.q` and `-q` legacy spellings are deleted with the rest of the legacy map entries,
per the phase brief; only `--q` stays tolerated.

## 2. `phase_question` becomes an ordinary phase sequence

`--code-0` only canonicalizes through `_split_phase_question_suffix`, so that splitter
must survive or the flag-Off branch and every historical `--code-N` row stop resolving.
Keep the mechanism, delete the question framing: rename it
`_split_phase_sequence_suffix` (and
`_canonical_plan_chain_suffix_without_phase_question` to `..._without_phase_sequence`),
and have it produce `kind="phase"` with `parent_suffix` set, instead of
`kind="phase_question"`. Role resolution is unchanged (`--code-0` stays `code`,
`--plan-0-0` stays `feedback`).

## 3. Two neutral replacements for the two exported question helpers

- `is_root_question_suffix` →
  `is_root_sequence_suffix(suffix, *, agent_family_role=None)`: true when the canonical
  suffix is a bare **numeric** root token (`--0`, `--2`) whose resolved role is absent
  or `root`. Numeric-only keeps `--reviewer` continuing as `--reviewer-@`, exactly as
  today; the `role in (None, "root")` clause keeps a legacy numeric feedback row (`--2`
  with role `feedback`) out of the root sequence, which is what the old `role == "q"`
  clause did.
- `question_followup_suffix_template` → `family_continuation_suffix_template`, same
  signature. Returns `--@` for a root-sequence suffix, `<parent>-@` for a phase-sequence
  suffix, and `<canonical>-@` otherwise, falling back to `--plan` when the suffix does
  not canonicalize. Every existing assertion in
  `tests/test_plan_chain_roles.py::test_question_followup_suffix_templates` and
  `tests/test_dynamic_agent_family_attach_directives.py` still holds under the new name.

## 4. The Off branch's continuation role becomes `root`, matching the On branch

The flag-On path already produces the target shape. `launch_gate_followup_agent` passes
`agent_family_role=policy.role or starter_role`, and the question gate shell declares no
`role`, so the follow-up inherits the starter's role — `root` for a promoted family root
(`agent/_family_promotion.py` writes `"root"`). So under the flag the asker's
continuation is already `<family>--0` with role `root`, and the TUI already
special-cases that (`compact_role_label` and `get_phase_label` both branch on
`agent_family_role == "root"` and render the bare suffix token).

The Off branch and the `%auto` short-circuit therefore pass `agent_family_role="root"`
where they passed `"q"`. Both branches then produce identical metadata, which is what
makes the three question-asker status predicates dead rather than merely unused.

Note that the `root_sequence` disjunct
`first_family_agent_question and not first_plan_agent_question` becomes redundant: that
case sets `interrupted_suffix = "--0"`, which `is_root_sequence_suffix` already matches.
Collapse it.

## 5. Question-asker status predicates are deleted, not rewritten

`has_inherited_family_question`, `is_answered_continuation_asker`, and
`is_answered_root_asker_step` each return early unless the row's role is `q`. Once no
row is written with role `q`, all three are unreachable, so "migrate the consumer to
read the gate shell instead" means: delete them and let the gate-shell row carry the
status. The question gate shell already declares `settled_status: "ANSWERED"` and a
`submit` branch status of `ANSWERED` in `question_shell/create.py`, so `ANSWERED` is
still on screen — on the gate shell member instead of on the asker.

`has_unanswered_completed_question` stays (it is `status-collapse`'s to delete) and its
`and not has_inherited_family_question(...)` guards simply drop: with no role-`q` rows
the guard is always `False`, so removing it is behaviour-preserving for every row this
change produces.

## 6. Two downstream fallbacks must be repaired, or the deletion regresses them

Both are direct consequences of `agent_family_role_for_suffix("--0")` returning `None`
instead of `"q"`, and both must land in this change:

- `agent/_family_attach_resolution.py::_family_role` falls through to
  `if suffix_arg == "@" or token.isdigit(): return "feedback"`. That branch is currently
  unreachable for numeric tokens (0/1 resolve to `q`, 2+ resolve to `feedback` earlier).
  After the deletion it starts firing for `--0`/`--1`, which would silently make a plain
  `%i(@, family=…)` fork a plan-**feedback** row — `"feedback"` is in
  `PLAN_CHAIN_MEMBER_ROLES`, so the whole family would start rendering as a plan chain.
  Return the bare token instead, matching the generic `return token` path just below.
- `ace/tui/models/_loaders/_meta_enrichment_common.py::apply_workflow_child_identity_from_meta`
  sets `agent.agent_family_role = agent_family_role_for_suffix(child_suffix)`, which
  drops a root's own `--0` step to no role at all. Pass the meta's stored
  `agent_family_role` through the existing role-aware parameter so the row keeps `root`.
  A `--plan` child suffix is unaffected: the stored role is only consulted on the
  root-token branch.

# Deliverables

1. **`src/sase/plan_chain.py`** — delete `PLAN_CHAIN_QUESTION_SUFFIX` (and its
   `_KNOWN_SUFFIXES` membership), the `.q`/`-q` legacy map entries, `"q"` from
   `_EXPLICIT_FAMILY_ROLES`, the `is_root_question` / `is_phase_question` properties,
   the `legacy_question` / `root_question` / `phase_question` kinds, and the
   `token in {"0", "1"}` inference. Rename the phase-sequence splitter and its
   canonicalization helper. Replace `is_root_question_suffix` with
   `is_root_sequence_suffix` and `question_followup_suffix_template` with
   `family_continuation_suffix_template`.
2. **`src/sase/axe/run_agent_exec_questions.py`** — update both
   `handle_questions_marker` (flag Off) and `_continue_after_auto_answered_question`
   (`%auto` short-circuit) to the new helpers and `agent_family_role="root"`; collapse
   the redundant `root_sequence` disjunct. Refresh the docstrings that describe the old
   suffix behaviour.
3. **`src/sase/agent/_family_attach_resolution.py`** — `_family_role` returns the bare
   token for an unclassified root-token suffix instead of `"feedback"`.
4. **`src/sase/ace/tui/models/_loaders/_meta_enrichment_common.py`** —
   `apply_workflow_child_identity_from_meta` passes the stored role through.
5. **`src/sase/ace/tui/models/_agent_status_family_policy.py`** — delete
   `has_inherited_family_question`, `is_answered_continuation_asker`,
   `is_answered_root_asker_step`.
6. **`src/sase/ace/tui/models/_agent_status_apply.py`** — drop both
   `not has_inherited_family_question(...)` guards and the `ANSWERED` override loop,
   plus the now-unused imports.
7. **`src/sase/ace/tui/models/_agent_status_family.py`** and
   **`_agent_status_overrides.py`** — drop the re-exports of the three deleted
   predicates.
8. **`src/sase/ace/tui/agent_context_members.py`** and
   **`src/sase/ace/tui/widgets/prompt_panel/_agent_display_content.py`** — drop the
   `PLAN_CHAIN_QUESTION_SUFFIX` label entries, the `"q"` entry in `_FAMILY_ROLE_LABELS`,
   and the `role == "q"` branches. Confirm by test that a historical `--q` row still
   labels `q` and still renders `AGENT (q)` through the generic suffix-token fallback.
9. **`src/sase/ace/tui/models/_agent_status_family_core.py`** — rewrite the comment that
   explains why role `q` is excluded from `PLAN_CHAIN_MEMBER_ROLES`; the exclusion of a
   role that no longer exists is not a reason.
10. **`src/sase/main/pipe_handler.py`** — drop `PLAN_CHAIN_QUESTION_SUFFIX` from
    `_RESERVED_PIPE_NAME_TOKENS`; `q` is no longer a phase name to collide with.
11. **`docs/ace.md`** — the two `--q` passages (the `QUESTION` propagation paragraph
    near the "next numeric phase" sentence, and the `AGENT (<role>)` phase-label list).
12. **Tests and goldens** — see below.

# Tests

Update, do not merely delete. Every test that pinned the question taxonomy should end up
pinning the ordinary-shell behaviour that replaced it.

Known-affected files (from grep on the deleted symbols and on `agent_family_role="q"`):

- `tests/test_plan_chain_roles.py` — rewrite the `--q`, root-question, phase-question,
  and follow-up-template cases. Add explicit coverage that `--q` still canonicalizes to
  `--q`, still splits as a family member, and now resolves to role `None`; and that `.q`
  / `-q` no longer canonicalize.
- `tests/test_dynamic_agent_family_attach_directives.py` — rename the
  `question_followup_suffix_template` import/use.
- `tests/test_dynamic_agent_family_attach_resolution.py` — the `("q", "q")` role-mapping
  case still passes (an explicit `q` suffix arg resolves to token role `q`); add a case
  pinning that an `@`-allocated root token does **not** resolve to `feedback`.
- `tests/test_axe_run_agent_exec_plan_followup_questions.py`,
  `tests/test_axe_run_agent_helpers_artifacts.py`,
  `tests/test_axe_run_agent_exec_finalize_metadata.py`,
  `tests/test_axe_run_agent_exec_plan_chat_paths.py` — flag-Off successor naming and the
  persisted role move from `q` to `root`.
- `tests/test_agent_loader_status_override_questions.py`,
  `..._question_continuations.py`, `..._question_families.py`,
  `..._question_runtimes.py`, `..._promoted_plan_family.py`, `..._followup_roots.py` —
  the deleted predicates and the `ANSWERED` override. Keep the cases that still describe
  real behaviour (gate-shell rows, unanswered `QUESTION` propagation); drop the ones
  that only existed for the `q` role.
- `tests/test_workflow_child_meta_enrichment.py` — the root's own `--0` step now
  enriches to role `root`.
- `tests/ace/tui/test_agent_context_members.py`,
  `tests/ace/tui/widgets/test_agent_display_phase_labels.py`,
  `tests/ace/tui/widgets/test_agent_display_output_variables.py`,
  `tests/ace/tui/widgets/test_prompt_panel_header.py`,
  `tests/ace/tui/test_skill_uses_loader.py`,
  `tests/ace/tui/test_opened_workspaces_loader.py`,
  `tests/ace/tui/test_memory_reads_loader_context.py`,
  `tests/ace/tui/test_artifact_reads_loader_context.py`,
  `tests/ace/tui/test_glossary_reads_loader_context.py` — replace the
  `PLAN_CHAIN_QUESTION_SUFFIX` import with the literal `"--q"` where the fixture only
  needs a historical suffix, and update the phase-label expectations for a role-`root`
  numeric row.

The list is the grep result, not a promise of completeness — run the suite and fix
whatever else moves.

# Verification

- `just install` first if the workspace is stale.
- `just check` (whole-repo lint gates plus the diff-scoped test lane). Symvision will
  catch any symbol left behind or newly orphaned by the deletions; read
  `sase memory read symvision:<keyword>` before adding a pragma rather than adding one.
- `just test-visual`. `tests/ace/tui/visual/_ace_agents_png_snapshot_fixtures.py` builds
  a fixture with `role_suffix="--q"` and `agent_family_role="q"`; the analysis says the
  label falls through to the same `q` text, so the PNG golden is expected **not** to
  change. If it does change, that is a real rendering regression to explain, not a
  golden to accept blindly.
- Because the change touches `plan_chain.py`, which most of the family/status machinery
  imports, hand `just check-full` to `/sase_monitor` with the `TESTING` / `TESTED`
  status pair before closing the bead.

# Risks and accepted consequences

- **Flag-Off rendering changes.** With `gate_shell_handoff` off, an answered question
  continuation renders `DONE` instead of `ANSWERED`, and its phase label reads
  `AGENT (0)` instead of `AGENT (q)`. That is precisely the flag-On rendering, and the
  Off branch is deleted in the next phase (`status-collapse`). Accepted.
- **Mid-flight Off-branch chains.** A question chain already on disk with role `q` and a
  suffix of `--2` will, after the upgrade, name its next continuation `--2-0` rather
  than `--3`, because `is_root_sequence_suffix` deliberately does not honour the deleted
  role. Beta flag, one round of exposure, no data loss. Accepted rather than
  reintroducing the role as a compatibility value.
- **`.q` / `-q` stop resolving.** Pre-`--` dotted and single-dash question spellings are
  no longer family-member suffixes, so an artifact dir that old drops out of family
  rendering. The phase brief scopes the compatibility promise to `--q` only.
- **Blast radius.** `plan_chain.py` is imported by the launcher, the runner, the pipe
  handler, and most of the TUI status machinery. The mitigation is `just check-full`,
  not a smaller diff — a partial deletion would leave the tree in a worse state than
  either end point.

# Out of scope

- Deleting the `gate_shell_handoff` Off branch, the flag, `_agent_status_overrides.py`,
  `_notification_status_overrides.py`, `has_unanswered_completed_question`,
  `planner_child_status`, the synthetic planner children, or the colour ladder. All of
  that is `status-collapse` (`sase-ud.13`).
- Memory notes, the decision record, and skill templates — that is `memory-and-skills`.
- Any change under `crates/sase_core`; the Rust core has no question-suffix taxonomy.

Record anything discovered but out of scope as
`sase bead note sase-ud.12 'PROPOSED FOLLOW-UP: …'` rather than creating a bead.
