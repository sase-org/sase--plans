---
tier: epic
title: Land cross-repo turns without stranding their epic or core pin
goal: 'A single agent turn that changes both sase and sase-core lands one green commit
  per repo with sase-core-revision.txt already pointing at the new core commit, agents
  know host commits happen after their turn so they close finished work instead of
  deferring it, and the stranded sase-1ck.4.1 landing is finished.

  '
phases:
- id: stranded_landing
  title: Move the core pin and finish the stranded sase-1ck.4.1 landing
  size: small
  depends_on: []
  description: 'stranded_landing: ratchet sase-core-revision.txt past 0541387 and
    1ad57ea, verify the +1 attachment tests, then close epic sase-1ck.4.1 and its
    parent phase sase-1ck.4 and mark note_cli.md done.'
- id: pin_follow
  title: Host moves a linked repo's revision pin when one declaration commits both
    repos
  size: medium
  depends_on: []
  description: 'pin_follow: add a repos.linked[].revision_pin config field, make the
    builtin commit finalizer commit pinned siblings first and write their pushed SHA
    into the primary''s pin file before the primary commit, with tests and docs.'
- id: after_turn_messaging
  title: Tell agents that host commits happen after the turn ends
  size: small
  depends_on: []
  description: 'after_turn_messaging: extend sase final submit output, the sase_final
    skill, and the bd/land_epic tale guidance so agents never defer a closeout waiting
    for their own host commit.'
proposed_by: bbugyi200.athena.0u6
create_time: 2026-09-29 17:21:17
status: done
bead_id: sase-1cq
---

- **PROMPT:** [prompts/202609/cross_repo_landing.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/cross_repo_landing.md)
- **BEAD:** [sase-1cq](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1cq/README.md)

# Land cross-repo turns without stranding their epic or core pin

## Why

Epic `sase-1ck.4.1` (bead note attachment CLI) is still `IN_PROGRESS`, and its parent
phase `sase-1ck.4` is still open, although its code has landed. Here is what happened:

1. The land agent `sase-1ck.4.1.land` found remaining work (`sase bead +1 -n` could not
   persist attachments). It wrote the completion tale
   `plan:202609/plus_one_note_attachments.md`. Following the `bd/land_epic` prompt, the
   tale ends with the epic closeout.
2. Tale step 3 said to move `sase-core-revision.txt` past the sase-core commit that
   holds the new wire/API. Steps 4-5 then closed `sase-1ck.4.1` and `sase-1ck.4`.
3. The coder `sase-1ck.4.1.land--1` changed both sase and sase-core in one turn.
   Completion is host-owned, so the new sase-core commit exists only after the turn
   ends. The coder could not move the pin, decided the closeout was blocked, and
   submitted `bead_action: keep` for both repos. It then checked `git log` again, saw
   nothing had landed, and resubmitted the same declaration twice (three accepted
   submissions in total). Its response says "host commits have not landed, so ...
   closing beads now would strand unlanded code."
4. After the turn, the host committed sase `6834fa1128` and then sase-core `0541387`, in
   that order. The finalizer commits and pushes the primary repo before any sibling.
   Nothing resumes a lander-authored tale, so the epic stayed open. The pin stayed at
   `43f744be`.
5. Cost: the Master Gate on `6834fa1128` fails
   `tests/test_bead/test_cli_attach_verbs.py::test_plus_one_note_attachments_persist`
   and `::test_plus_one_note_attachments_on_snooze_wake`, because CI builds the old
   core.

This is not a one-off. Prompt-prediction work landed sase `9d60b97513` (14:48) five
minutes before its core commit `1ad57ea` (14:53) from the same turn. The lint job's
"Check pinned core bindings" step has failed since then on
`evaluate_prompt_prediction_replay`. Many epic plans spend a whole phase only to "bump
the core pin". The only automatic bump is `.github/workflows/core-pin-ratchet.yml`,
which runs every six hours and opens a PR that nobody auto-merges.

Root causes this epic fixes:

- **No way to pin a same-turn core commit.** One declaration that commits both sase and
  sase-core can never carry the new core SHA in sase's pin. The host commits sase first,
  and the SHA does not exist during the turn (phase `pin_follow`).
- **Nothing tells agents that commits happen after the turn.** `sase final submit`
  prints only `Accepted final declaration for: commit`, and `bd/land_epic` never says
  that a tale's closeout must not wait for the tale's own commit (phase
  `after_turn_messaging`).
- **The stranded epic itself** (phase `stranded_landing`).

All three phases are independent and can run in parallel.

## Phase `stranded_landing` — move the core pin and finish sase-1ck.4.1

Finish the closeout that `plan:202609/plus_one_note_attachments.md` steps 3-5 asked for
and `sase-1ck.4.1.land--1` skipped.

1. Move the pin with `just ratchet-core-revision`, which moves it to sase-core's remote
   HEAD. Confirm that the new SHA contains both `0541387` (+1 evidence attachments) and
   `1ad57ea` (the `evaluate_prompt_prediction_replay` binding):
   `git merge-base --is-ancestor` in the linked checkout opened with
   `sase repo open sase-core`. Rebuild the binding with `just rust-install` and run
   `.venv/bin/python tools/check_sase_core_rs_bindings`. Then run the focused tests
   `tests/test_bead/test_cli_attach_verbs.py` and the bead note/+1/presentation tests,
   then `just check` through `sase tool run`. Do not run `just check-full`. Pre-existing
   reds are already tracked: `sase-1cm`, `sase-1cn`, and the parent-epic terminology
   audit on `sase-1ck`. Report them rather than fixing them.
2. Read `sase-1ck.4.1` and its three closed phases. Confirm that `sase bead +1 -n`
   persists attachments on the landed code, and confirm that
   `sase bead epic-symbols sase-1ck.4.1` is empty; resolve or re-key any entry per the
   Symvision epic-whitelist policy. Then run
   `sase bead close sase-1ck.4.1 --note "<verification, including the pin move and the land--1 keep>"`
   and `just symvision`. Set `status: done` in the frontmatter of
   `plan:202609/note_cli.md` through the plans repository opened with `sase repo open`.
   Never use `--force`.
3. Parent `sase-1ck.4` is a phase bead. Verify that plan `note_cli.md` fulfilled its
   note/close/update/+1/TUI/attach/list/path/text-rendering scope, then close only that
   phase: `sase bead close sase-1ck.4 --note "<what was verified>"`. Leave the
   containing epic `sase-1ck` open for its own land agent.
4. Declare the pin change as a normal `commit`, for example
   `chore(core-pin): ratchet sase-core-revision.txt to <sha12> for +1 attachments and prompt-prediction replay`.

## Phase `pin_follow` — the host moves a declared revision pin

Goal: when one accepted commit declaration commits both the primary checkout and a
linked repo whose config declares a revision pin, the host commits and pushes that
linked repo first. It then writes the new pushed SHA into the pin file, so the primary
commit carries the matching pin. That makes the whole change one green commit per repo,
with no follow-up turn. This is a project config field, not a feature flag: a project
chooses it permanently.

### Config

1. Add an optional `revision_pin` field to `repos.linked[]` entries. It is a path
   relative to the primary checkout root and names a file that holds one full
   40-character SHA of that linked repo. Parse and normalize the field in
   `src/sase/_linked_repo_config.py` and expose it on the linked-repo model
   (`src/sase/_linked_repo_env.py` and the repo inventory models that carry
   `auto_clone`). Reject absolute paths and paths that leave the checkout. Add a
   `sase doctor` repo-config check (`src/sase/doctor/checks_config_repos.py`) that flags
   a declared pin file that is missing or does not hold a 40-hex SHA.
2. Set `revision_pin: sase-core-revision.txt` on the `sase-core` entry in
   `sase/sase.yml`.

### Finalizer behavior (`src/sase/finalizers/`)

The main files are `commit_execution.py`, `commit_declaration.py`
(`dirty_repos_in_context_order`), `commit_dispatch.py` (`dispatch_commit_decisions`),
and `commit_repair_stitch.py`.

3. **Ordering.** When the accepted declaration commits the main repo and one or more
   pinned siblings, dispatch those pinned siblings before the main repo. Every other
   repo keeps its current context order, and declarations without a committed pinned
   sibling keep today's order exactly.
   - Put the reorder in one helper used by all three `dirty_repos_in_context_order` call
     sites in `commit_execution.py` (initial, checkpoint recovery, already-clean
     resume).
   - Keep conflict repair's remaining-repo handoff in `commit_dispatch.py` consistent
     with the new order.
   - The main repo's stitch still applies `bead_action`. The assigned bead now closes
     only after the pinned sibling has landed.
4. **Pin write.** After a pinned sibling's stitch succeeds, take its pushed `commit_sha`
   evidence. Write `<sha>\n` to the main checkout's pin file only when all of these
   hold:
   - the main repo's decision is `commit`, not a deferral;
   - the SHA is reachable from the sibling's remote default branch, fetching if needed;
   - the main checkout's current pin value is an ancestor of the SHA, so the pin never
     moves backward or sideways;
   - the file does not already hold that SHA.

   If any condition fails, skip the write and record a finalizer diagnostic. A skipped
   pin never fails the run.

5. **Host-authored path.** The main stitch then commits the pin change together with the
   agent's work, keeping the agent's commit message. Make the pin path host-authored
   for:
   - the stale-obligation, unexpected-path, protected-path, and
     `reject_discarded_dirty_work` checks;
   - checkpoint and unpushed-resume recovery of the main stitch. Rewriting the pin is
     idempotent, and an already-written pin is expected state, not stale dirt.

   Record finalizer evidence `revision_pin` = `<sibling>:<pin path>:<old12>-><new12>`
   (or the skip reason). If a concurrent pin change on origin conflicts during the main
   stitch's sync, leave it to the existing conflict-repair flow; add no special
   resolver.

### Tests and docs

6. Tests (extend the existing finalizer commit-dispatch and commit-execution suites):
   - pinned sibling ordered first, and today's order unchanged without a pin;
   - the pin written and included in the main commit, with evidence recorded;
   - each skip condition: main deferred or not dirty, SHA not on the default branch, pin
     not an ancestor, already equal;
   - recovery resume idempotency;
   - `bead_action: close` applied after the sibling lands;
   - config parsing, validation, and the doctor check.
7. Docs:
   - Update `docs/rust_backend.md` "The CI source revision pin". Replace "bump
     `sase-core-revision.txt` alongside that change once the `sase-core` commit is
     pushed" with the host-owned rule: a declaration that commits both repos gets the
     pin automatically. Agents only bump the pin by hand when their sase change needs an
     already-landed core commit (`just ratchet-core-revision`).
   - Document `revision_pin` in `docs/configuration.md` next to the other `repos.linked`
     fields.
   - Document the ordering and pin write in `docs/commit_workflows.md` where the builtin
     commit finalizer is described.
   - Do not edit SASE memory. The core memory line saying the pin must move past the
     core commit stays accurate, and it points at `docs/rust_backend.md`.

## Phase `after_turn_messaging` — make the after-turn commit timing explicit

1. `src/sase/main/final_handler.py` (`_handle_submit`): when the accepted declaration
   includes a `commit` payload, print a second line after
   `Accepted final declaration for: ...`. It says the host commits the declared
   repositories after this turn ends: until then they stay dirty and `git log` is
   unchanged, so end the turn now without re-checking or resubmitting. When the accepted
   payload carries a `bead_action`, name it. Say `close` closes the assigned bead after
   the primary commit lands, and `keep` leaves it open with nothing resuming it. Update
   the final-handler tests that assert submit output.
2. `src/sase/xprompts/skills/sase_final.md`, step 5: add the same point in one or two
   sentences. The host commits after the turn ends, so the repos stay dirty and
   `git log` is unchanged. Choose `bead_action` from the work's completeness, never from
   whether commits are visible yet. Read `sase memory read generated_skills.md` first,
   and regenerate and deploy the skill the way it prescribes.
3. `src/sase/default_config.yml`, `bd/land_epic`: after "a lander-authored tale must
   finish the landing itself", add guidance that the tale's coder commits only after its
   turn ends. So the closeout must never wait for, or be ordered after, a step that
   needs this work's own commit (its SHA, push, or CI result). Closing the epic in the
   same turn as the final code is the normal landing. Keep the wording project-agnostic,
   and update any snapshot or golden test that pins this prompt text.

## Verification

- Each phase runs `just fmt`/`just fix` and then `just check` through `sase tool run`.
  Nobody runs `just check-full`.
- `stranded_landing`: `sase bead read sase-1ck.4.1` and `sase-1ck.4` show `CLOSED`;
  `note_cli.md` has `status: done`; the pin contains `0541387` and `1ad57ea`.
- `pin_follow`: the unit tests above, plus one end-to-end finalizer test with a fake
  primary and a fake pinned sibling. It must show the sibling commit first and the
  primary commit's `sase-core-revision.txt` equal to the sibling's pushed SHA.
- `after_turn_messaging`: submit-output tests cover a commit payload with `keep` and
  with `close`, and the regenerated `sase_final` skill contains the new sentence.
