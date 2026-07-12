---
create_time: 2026-07-11 08:36:02
status: done
prompt: 202607/prompts/confirm_github_sdd_creation.md
tier: tale
---
# Confirm GitHub SDD Companion Repository Creation

## Goal

Require an explicit, default-no `y/n` confirmation immediately before `sase sdd init` creates a missing GitHub SDD
companion repository. The same protection must apply through the `sase init sdd` compatibility alias and when the SDD
initializer is selected by bare `sase init`, including `sase init --yes`. Existing companion repositories, local and
in-tree SDD stores, read-only checks, and non-SDD materialization call sites should retain their current behavior.

The confirmation must be based on authoritative provider discovery, not the absence of a local clone or store record: a
GitHub companion can already exist remotely even when this machine has never materialized it.

## User-visible behavior

- Preserve the current project validation and `is_sase_managed` guard before any provider work.
- Keep `sase sdd init --check` fully read-only and non-interactive. It may continue to report the conservative "create
  or connect" action without probing GitHub.
- During an applying init, let the GitHub provider perform a read-only `gh repo view` probe first.
  - If the configured/default companion exists, label, clone, adopt, import, and refresh it without the new creation
    prompt.
  - If the companion is definitively absent, prompt once with the host, full repository name, and public visibility, for
    example: `Create public GitHub SDD companion repository acme/widget--sdd on github.com? [y/N]`.
  - Accept only `y` or `yes`, case-insensitively. Blank input and every other answer decline.
- Treat the creation confirmation as non-bypassable. In particular, bare onboarding's generic `Run sase init sdd now?`
  approval and `sase init --yes` do not authorize creation of a new GitHub repository. When bare interactive onboarding
  reaches an absent companion, the resource-specific prompt may therefore follow the coordinator's generic initializer
  prompt.
- On a negative answer, EOF/non-interactive stdin, or interruption, print a concise cancellation/error message, return a
  nonzero result because SDD initialization remains incomplete, and perform no repository creation, label mutation,
  clone, store-record write, legacy import, generated-guide write, commit, or push.
- Do not add a `--yes` flag to `sase sdd init`. Document that unattended initialization can connect an existing
  companion but cannot authorize creation of a missing one.

## Technical design

1. Add a read-only SDD companion preflight capability to the workspace-provider contract in `sase`. It should return a
   small structured result that distinguishes `found`, `not_found`, and `unavailable` and identifies the provider, host,
   full repository name, and intended visibility. Document and test that the hook may perform network reads but must not
   create or label repositories, clone, or write local state.
2. Route explicit SDD initialization through that preflight before provider materialization when the selected storage
   policy is `separate_repo`. Fail closed if the provider cannot supply an authoritative preflight result, so version
   skew cannot silently fall back to the old auto-create behavior. Keep local/in-tree providers and all early
   unmanaged/invalid/non-project exits outside this path.
3. Add an explicit per-invocation creation authorization to the provider materialization options. Authorization is
   granted only after the CLI receives `y/yes`; a `found` preflight carries no creation authorization. This preserves
   the invariant under races: if a repository found during preflight disappears before materialization, the provider
   must stop instead of recreating it; if another actor creates an approved missing repository first, normal
   create-or-adopt recovery may connect to it.
4. Implement the preflight and authorization enforcement in `sase-github` by reusing its existing origin parsing,
   companion-name override, enterprise-host, and detailed repository-probe helpers. Move no GitHub naming or API
   knowledge into the `sase` CLI. Immediately before each `_create_github_sdd_repo` branch, require the explicit
   authorization supplied by the init flow; keep existing-repository labeling and cloning behavior unchanged.
5. Scope the new mandatory prompt to the explicit SDD initializer path. Other existing SDD materialization consumers
   (workspace setup, agent/bead/plan flows, prompt export, and TUI actions) retain their current provider-owned behavior
   unless they opt into the new guarded preflight contract. Make this distinction explicit in API documentation and
   tests.
6. Coordinate the host/plugin compatibility boundary. Update the GitHub plugin's minimum supported `sase` version if it
   consumes new public provider types or hooks, and ensure a new host paired with an older plugin reports an update
   requirement before materialization rather than risking unconfirmed repository creation.

## Tests and documentation

- In `sase`, add focused handler/provider-contract coverage for affirmative, negative, blank, EOF/non-TTY, and
  interrupted confirmation; exact `--path` targeting; compatibility alias dispatch; bare onboarding with and without
  `--yes`; and fail-closed behavior when preflight support is unavailable.
- Assert ordering and side-effect boundaries: management validation precedes preflight; prompt follows a definitive
  `not_found`; materialization and generated-file initialization run only after approval; `--check`, unmanaged, invalid,
  existing-record, local, and in-tree paths do not prompt.
- In `sase-github`, cover default and overridden repository names, GitHub Enterprise hosts, existing and missing
  companions, approval and denial, unavailable/auth/network probes, missing authorization, create/adopt races, and the
  guarantee that denial issues no `gh repo create`, label, or clone command.
- Update `sase sdd init` and `sase init sdd` help text plus the SDD/initialization/configuration guides to describe the
  exact prompt, default-no and non-interactive behavior, and the fact that bare `--yes` cannot approve GitHub repository
  creation. Update the GitHub plugin README, configuration guide, and architecture hook table with the same contract.
- Run each repository's focused tests while developing, then its formatter/lint/type gates and full required check
  (`just check` in both repositories after installing their workspace dependencies as required).

## Acceptance criteria

- No execution of `sase sdd init`, its compatibility alias, or its bare-onboarding route can create a missing GitHub SDD
  companion without a fresh affirmative `y/yes` response naming that repository.
- A declined or unavailable confirmation leaves both GitHub and the local project unchanged and returns nonzero with a
  clear explanation.
- An already-existing companion is connected and refreshed without a repository-creation prompt.
- `--check` remains side-effect-free and does not require stdin or network access.
- Version skew and discovery races fail closed; they never restore silent auto-creation on the guarded CLI path.
- Existing non-CLI materialization workflows and non-GitHub storage modes preserve their documented behavior.
