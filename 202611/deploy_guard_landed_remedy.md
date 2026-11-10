---
tier: tale
title: Point bob plugin deploy-guard refusals at the landed-build remedy
goal:
  When `bob plugins sync` refuses to overwrite another checkout's deploy, the refusal
  and the bob-plugins agent guidance lead agents to fetch, reconcile from a clean
  checkout without `-F`, and ask Bryan only when the protected build is unlanded and
  abandoned.
size: medium
proposed_by: bbugyi200.athena.1as
proposed_with: claude/opus
create_time: 2026-11-10 07:19:03
status: wip
---

# Plan: Point bob plugin deploy-guard refusals at the landed-build remedy

## Background

The bob-cli research report `202611/j8_6_1_1_deploy_guard_question.md` (in the
`bob-cli--research` sidecar) covers an incident on 2026-11-09. Agent `bob-cli-j8.6.1.1`
ran `bob plugins sync --no-pull` from a checkout last fetched at 15:18. The guard
correctly refused to overwrite `bob-ledger-tools`, because another agent
(`bob-cli-j8.7`) had deployed an uncommitted build. That build had been committed as
`fe7097f` 11 minutes earlier, with identical bytes. The refusal offered only two
remedies, a dry-run preview or `-F`, so the agent asked Bryan "keep protected or force?"
and waited about 1h44m. The "force" answer rolled the vault back from ledger-tools
1.102.0 to 1.101.1 and dropped j8.7's notebooks API. The documented reconcile path
(fetch, update to the landed commit, sync from a clean tree) would have needed neither
`-F` nor a question.

The report makes three recommendations. Each one was checked against the current code
and live state:

1. **Repair the vault.** Already done; nothing to implement. The live
   `~/bob/.obsidian/plugins/bob-ledger-tools` holds manifest `1.103.0`, with a `clean`
   deploy stamp at `254b4c4`. `fe7097f` is an ancestor of `254b4c4`, and the deployed
   `main.js` carries the notebooks API. `bob-navigation-hotkeys` is `2.77.0`. This plan
   only re-checks it (step 4).
2. **Teach the guard the third remedy** (bob-cli `src/native/plugins/deploy.rs`).
   Agreed. `refuse_message` (around line 1279) prints only the preview and `-F`
   commands. `find_inclusion_proof` walks only `HEAD`. There is one refinement: in the
   incident the agent's remote-tracking refs were also stale (fetched at 15:18,
   `fe7097f` pushed at 15:31). A hint that only names a proving commit on
   already-fetched refs would not have fired. So the refusal must also carry a generic
   "fetch, update, sync from a clean tree, no `-F`" remedy when no landed commit is
   found locally. The same stale-checkout trap also hits the clean-stamp "incoming HEAD
   does not contain the stamped clean revision" refusal, so that refusal gets the hint
   too. The stamp already records the deploying `agent` (from `SASE_AGENT_ID`) but the
   refusal never prints it. Printing it lets an agent check whether the author is still
   running.
3. **Tighten agent guidance** (bob-plugins `AGENTS.md` / `CLAUDE.md` /
   `docs/develop/deployment.md`). Agreed. bob-plugins `AGENTS.md` is a short
   hand-maintained file (no `sase/memory`). `CLAUDE.md` is a near-copy with different
   line wrapping. Neither covers deploy refusals. `deployment.md` also understates the
   guard: it says a reconcile needs "a clean checkout of the same bytes", but bob-cli
   accepts any clean checkout whose HEAD reaches a commit that holds the deployed bundle
   byte for byte.

## Repository access

All edits land in two repos outside a sase workspace. Open them with the `/sase_repo`
skill and work only in the printed paths:

- `sase repo open bob-cli -r "<reason>"` opens the registered bob-cli project checkout.
- `sase repo open gh:bobs-org/bob-plugins -r "<reason>"` opens bob-plugins. Inside the
  bob-cli project its configured linked name `bob-plugins` resolves to the same repo.

Read each repo's `AGENTS.md` before editing. Bring both checkouts up to date with
`git pull --ff-only` first. Read bob-cli's reference memory with
`sase memory read docs.md cli_rules.md -p bob-cli -r "<why>"`. Both repos become commit
obligations for the final declaration. Suggested subjects:
`feat(plugins): name the landed build in deploy-guard refusals` (bob-cli) and
`docs(deployment): reconcile landed builds before asking to force` (bob-plugins). Per
bob-cli's release-train decision, do not edit `CHANGELOG.md` or the `Cargo.toml`
version. This is not a contract break: the refusal is an error string, and every
substring the existing tests assert stays.

## Step 1: bob-cli guard refusal (`src/native/plugins/deploy.rs`)

### 1a. Landed-build lookup (advisory, local refs only)

Add a helper, used only on the refusal path, that looks for the protected build among
**already-fetched remote-tracking refs** (`refs/remotes/*`):

- **Dirty stamp from another checkout or host:** walk
  `git rev-list --max-count=500 --remotes --not HEAD -- plugins/<id>`, newest first.
  Find the first commit whose tree holds every stamped asset byte for byte. Refactor
  `find_inclusion_proof` so the HEAD walk and the remotes walk share one implementation
  (the same `commit_holds_stamp` + `BatchHasher` session, parameterized by the rev-list
  revision arguments) instead of duplicating it. Per-file matches from different commits
  still never qualify.
- **Clean stamp whose `source_head` is not an ancestor of HEAD:** if that object exists
  locally and some remote-tracking ref contains it, the deployed revision has landed.
- **Naming the ref:** list containing refs with
  `git for-each-ref --contains <sha> --format=%(refname) refs/remotes`. Drop symbolic
  `refs/remotes/*/HEAD`. Prefer the current branch's upstream
  (`git rev-parse --abbrev-ref --symbolic-full-name @{upstream}`) when it contains the
  commit; otherwise use the first remaining ref in sorted order. Print the short name
  (for example `origin/master`).
- Never run `git fetch` or anything else that touches the network or mutates refs. Set
  `GIT_TERMINAL_PROMPT=0` as the existing git calls do. Treat any git failure as "not
  found", which falls back to the generic remedy.
- **The guard decision does not change.** A landed commit that HEAD does not reach still
  refuses, because deploying the stale tree would roll the vault back. The lookup only
  shapes the message, and permit paths pay no extra cost.

### 1b. Refusal message (`refuse_message`)

- When `stamp.agent` is set, name it:
  `... at <root> on <host> by agent <id> (<sha>, <time>)`.
- For the **dirty-from-another-checkout** refusal and the **clean non-ancestor**
  refusal, insert a remedy sentence before the preview/force commands:
  - Landed commit found (dirty stamp): say that commit `<short>` on `<ref>` holds the
    deployed bundle byte for byte, so that build has landed. Then: stash your changes,
    bring this checkout up to `<ref>`, sync from the clean tree (no `-F` needed), then
    restore and redeploy your changes.
  - Landed commit found (clean stamp): say the deployed revision `<short>` is on
    `<ref>`, so bring this checkout up to `<ref>` and sync again; no `-F` needed.
  - Not found: if that build has since landed, run `git fetch`, bring this checkout up
    to date, and sync from a clean tree; no `-F` needed.
- Reword the force clause so it reads as the last resort and states its cost, for
  example "overwrite, discarding the deployed build, with
  `bob plugins sync --repo … -p <id> -F`".
- Keep the substrings existing tests assert: `refusing to overwrite plugin <id>`,
  `deployed from uncommitted build`, `--dry-run`, `-F`.
- All other refusals (same-checkout divergent history, pending stamp, digest mismatch,
  unknown provenance, malformed, unsupported) keep their current remedies. Pass the
  optional remedy sentence into `refuse_message` instead of special-casing it inside.
- Keep the message one line, in the existing style. The exact wording is the
  implementer's call within these constraints.

## Step 2: bob-cli tests (`tests/cli/plugins.rs`)

Build the fixtures on a **bare** upstream so checkouts can push and fetch. Reuse the
existing upstream helpers (`advance_plugins_remote` and friends) where they fit; extend
`init_two_checkouts_and_vault` or add a sibling fixture. Set `SASE_AGENT_ID` explicitly
where a test asserts on it, and `env_remove` it elsewhere, so a test run inside a SASE
agent cannot leak an agent id into stamps.

1. **Landed dirty build is named.** A dirty-deploys `alpha` with
   `SASE_AGENT_ID=agent-a`, then commits and pushes. B fetches only. B's sync exits 1.
   Stderr names A's commit (short SHA), an `origin/` ref, `agent-a`, and says no `-F` is
   needed. Vault bytes and stamp are unchanged.
2. **Documented recovery works end to end** (continue from 1). B has its own uncommitted
   edit. It runs `git stash push -u`, `git merge --ff-only origin/<branch>`, then a
   clean sync, which succeeds and leaves a `clean` stamp at A's commit. Then
   `git stash pop` and a dirty sync of B's change, which succeeds without `-F`. The
   vault now holds B's bytes.
3. **Generic remedy, and no fetch.** A pushes; B does not fetch. The refusal contains
   `git fetch` and does not contain A's SHA. B's `refs/remotes/origin/<branch>` is
   unchanged afterwards.
4. **Clean-stamp non-ancestor.** A commits, clean-syncs, and pushes. B fetches only; its
   HEAD is older and its bytes differ. The refusal names A's commit on `origin/…` and
   says no `-F` is needed.

`plugins_sync_refuses_dirty_deploy_from_another_checkout` and
`plugins_sync_committed_dirty_deploy_overwrites_without_force` must keep passing
unchanged, or with strictly added assertions. Add a unit test in `deploy.rs` if the
ref-preference logic becomes a separate pure function.

## Step 3: bob-cli docs (`docs/plugins.md`)

Behavior-contract updates only (per bob-cli `docs.md` memory):

- Replace "A protection refusal names the plugin and earlier deployment, explains the
  cause, and prints the preview and force commands." with text saying the refusal names
  the plugin and the earlier deployment (including the deploying agent when recorded),
  explains the cause, and gives the landed-build remedy before the preview and force
  commands. That remedy names a commit on an already-fetched remote-tracking ref that
  holds the deployed build, or otherwise tells you to fetch.
- Next to "Protection never depends on a remote fetch.", state that the refusal's
  landed-build hint reads only remote-tracking refs already present, never fetches, and
  never changes the decision.
- In the recovery paragraph ("To recover an uncommitted deploy that has since been
  committed…"), add the stash → update → clean sync → restore → redeploy sequence for a
  checkout that has its own uncommitted work.
- No clap help change is needed because guard behavior is unchanged. If `long_about` is
  touched anyway, regenerate with `just docs-gen`.

## Step 4: bob-plugins guidance

- **`AGENTS.md` and `CLAUDE.md`:** add the same paragraph to both. Keep the content
  identical; only wrapping may differ, as today. It says: if `bob plugins sync` refuses
  to overwrite another checkout's deploy, never pass `-F` on your own and do not ask
  Bryan yet. Run `git fetch` and `git log --oneline HEAD..@{upstream} -- plugins/<id>`.
  - If the protected build has landed (the refusal names the commit when it is already
    fetched), follow "When sync refuses an overwrite" in `docs/develop/deployment.md`.
    No `-F`, no question.
  - If it has not landed and the agent named in the refusal is still running
    (`sase agent list`), wait for it to land instead of forcing.
  - Ask Bryan only when the build is unlanded and abandoned. The question must name what
    `-F` would remove from the vault: the deployed plugin version and the features or
    APIs it carries.
- **`docs/develop/deployment.md`:**
  - Correct "After an uncommitted fix is committed and present in a clean checkout of
    the same bytes, a real sync reconciles the stamp without `--force`". The accurate
    rule: any clean checkout whose HEAD reaches a commit holding the deployed bundle
    byte for byte syncs without `--force`, and the sync names that proving commit.
  - Add a `### When sync refuses an overwrite` subsection with the three-way procedure
    (landed → self-serve; author still running → wait; unlanded and abandoned → ask,
    naming what `-F` drops) and this recipe:

    ```bash
    git fetch
    git log --oneline HEAD..@{upstream} -- plugins/<id>
    git stash push -u
    git merge --ff-only @{upstream}   # rebase instead if you have local commits
    bob plugins sync --repo "$PWD" --no-pull -p <id>
    git stash pop
    npm run build                     # source-fragment plugins
    bob plugins sync --repo "$PWD" --no-pull -p <id>
    ```

    Explain why it works: the clean sync rewrites the stamp to the landed commit, so the
    second sync is a descendant deploy and the vault keeps both changes. `-F` is
    appropriate only when Bryan approved dropping the deployed build, or when the
    deployed bytes are in no commit (for example after an earlier force) and you are
    deploying a clean commit that contains the work they carried.

- **Vault re-check (recommendation 1, read-only):** confirm the vault's
  `bob-ledger-tools` manifest is `1.102.0` or later. Confirm its `.bob-deploy.json`
  stamp is `clean` at a commit for which
  `git merge-base --is-ancestor fe7097f <stamp head>` succeeds. Do not redeploy. If it
  has regressed, stop and report; never force.

## Verification

- bob-cli: `just check` (fmt, clippy, all tests) and `just docs-check`. If a recipe
  refuses raw agent runs, use the wrapper it names. Run `cargo test --test cli plugins`
  first for fast iteration.
- bob-plugins: `npm run docs:check` and `npm run validate`. Because `AGENTS.md` requires
  a sync after any change, run `bob plugins sync --repo <checkout> --no-pull --dry-run`.
  Run the real sync only if every row is unchanged or synced. If it refuses or shows
  drift, do not pass `-F`; report it.
- Do not install or release bob-cli. The new message reaches agents through the normal
  release train.

## Out of scope

- Guard decisions stay as they are: no auto-fetch, and no permit based on a proof that
  exists only on remote refs.
- `sase repo open` skips dirty linked checkouts entirely: no fetch, no fast-forward, no
  "behind upstream" notice (`refresh_clean_linked_checkout` in sase). That is why the
  resumed agent also synced from the stale base. It is a possible sase follow-up, not
  part of this plan.
