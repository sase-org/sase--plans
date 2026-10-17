---
tier: tale
title: Create the Step 0 master-integrity ruleset
goal:
  sase-org/sase master rejects force pushes and deletion through one active repo ruleset
  with no bypass actors.
size: xsmall
proposed_by: bbugyi200.apollo.8t
proposed_with: claude/opus
create_time: 2026-10-17 18:52:09
status: wip
---

# Create the Step 0 `master-integrity` ruleset on `sase-org/sase`

## Goal

Apply the merge-queue fence reassessment's **Step 0**. Add one repo-level branch ruleset
to `sase-org/sase` that blocks force pushes to `master` and blocks deleting `master`.
Nothing else changes.

The maintainer asked for this to be done from the command line. Approving this plan is
the human sign-off that requirement N11 asks for ("rulesets go through a human gate;
agents never apply them unasked"). Do **not** open another `/sase_gate` for the same
change. Run the `gh api` commands below directly.

## Facts checked while planning (2026-10-17)

- `sase-org/sase` is **public**, and the default branch is `master`.
- The active `gh` account (`bbugyi200`) has `admin: true` on the repo. Its token has the
  `repo` scope, which is enough to create a repo ruleset.
- `gh api repos/sase-org/sase/rulesets` returns `[]`, and
  `gh api repos/sase-org/sase/rules/branches/master` returns `[]`.
- `gh api repos/sase-org/sase/branches/master/protection` returns 404 ("Branch not
  protected"). There is no classic protection.
- The org is on the Free plan, so org-level rulesets return 403. The ruleset **must** be
  created at the repo level.
- Nothing in sase force-pushes `master`:
  - `.github/workflows/core-pin-ratchet.yml` force-pushes only its own
    `core-pin-ratchet` branch.
  - release-please updates only `release-please--branches--master`.
  - The `push --force --all` strings in `src/sase/bead/cli_attachment_*.py` are printed
    manual instructions for the separate attachments repo.

  Ordinary fast-forward landings are therefore unaffected.

## Scope

In scope: exactly two rules, `deletion` ("Restrict deletions") and `non_fast_forward`
("Block force pushes"), targeting only `refs/heads/master`.

Out of scope (do **not** add, even though the API makes it easy):

- Merge queue, linear history, required PR, and required status checks. These belong to
  Epic D (D2/D5). Turning any of them on today would refuse the agents' direct pushes to
  `master`.
- The `v*` tag ruleset. It waits, per N12.
- Any bypass actor. N9 says these two protections are never relaxed, so the maintainer
  must not be added either.
- An "all branches" or wildcard target. release-please force-updates its own release-PR
  branch, and that must keep working.
- Classic branch protection, used as a fallback if the ruleset call fails.

There are no repository file changes, so this run has nothing to commit.

## Steps

1. **Re-check the "before" state.** Make this idempotent and safe:

   ```bash
   gh api repos/sase-org/sase/rulesets
   gh api repos/sase-org/sase/rules/branches/master --jq '[.[].type]'
   ```

   - If both print `[]`, continue to step 2.
   - If a ruleset named `master-integrity` already exists, do **not** create a second
     one. Fetch it with `gh api repos/sase-org/sase/rulesets/<id>`. If it already
     matches the spec in step 2, skip to step 3. If it differs, stop and report the
     difference without changing it.
   - If any other ruleset exists, stop and report it. Do not change it. Someone else
     changed the repo after this plan was written.

2. **Create the ruleset.** Use exactly this payload:

   ```bash
   gh api -X POST repos/sase-org/sase/rulesets --input - <<'EOF'
   {
     "name": "master-integrity",
     "target": "branch",
     "enforcement": "active",
     "bypass_actors": [],
     "conditions": {"ref_name": {"include": ["refs/heads/master"], "exclude": []}},
     "rules": [{"type": "deletion"}, {"type": "non_fast_forward"}]
   }
   EOF
   ```

   Record the returned `id`. If the call fails (for example 403, 404 or 422), stop and
   report the full error output. Do not retry with a different payload, at the org
   level, or through classic branch protection.

3. **Verify by reading the rules back.** This is the reassessment's only sanctioned
   check:

   ```bash
   gh api repos/sase-org/sase/rules/branches/master --jq '[.[].type] | sort'
   # expect exactly: ["deletion","non_fast_forward"]
   gh api repos/sase-org/sase/rulesets --jq '.[] | {id, name, enforcement, target}'
   # expect exactly one entry: master-integrity, "active", "branch"
   gh api repos/sase-org/sase/rulesets/<id> \
     --jq '{bypass_actors, conditions, rules: [.rules[].type]}'
   # expect: bypass_actors == [] (or null/absent),
   #         conditions.ref_name.include == ["refs/heads/master"], exclude == [],
   #         rules == ["deletion","non_fast_forward"]
   ```

   If any check fails, report the exact output. Do not "fix" it by adding rules or
   bypass actors. To undo a wrong ruleset, delete it with
   `gh api -X DELETE repos/sase-org/sase/rulesets/<id>`, but only when the read-back
   shows the created ruleset is wrong in a way that could block normal fast-forward
   landings, such as an extra rule or a wider target.

4. **Do not test destructively.** Never run `git push origin :master`, a force push, or
   any branch-delete call against `sase-org/sase`. If the ruleset were wrong, such a
   test could delete or rewrite production `master`. Destructive tests belong to Epic
   C's sandbox repo.

## Done when

- `gh api repos/sase-org/sase/rules/branches/master` lists exactly `deletion` and
  `non_fast_forward`.
- The only ruleset on the repo is `master-integrity`: `active`, branch target,
  `refs/heads/master` only, and an empty bypass list.
- No repository files changed and no destructive test was run.

## Report back to the maintainer

In the final response, include the ruleset `id`, the read-back output, and these two
human follow-ups. This turn cannot wait on either of them:

- **Watch the next ordinary landing.** The next agent fast-forward push to `master` and
  the Master Gate run it triggers should go through unchanged. If a push is refused with
  a ruleset error, delete the ruleset (command in step 3) and investigate.
- **Leave this ruleset alone when the queue arrives.** At D2, put the merge-queue rules
  and the maintainer's pilot bypass in a **separate** ruleset. A bypass actor is exempt
  from every rule in its ruleset. Adding the queue rules and the bypass to
  `master-integrity` would therefore quietly relax force-push and deletion protection,
  which N9 forbids.
