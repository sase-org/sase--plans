---
tier: tale
title: Exclude Obsidian area-note repos from chezmoi routine jobs
goal:
  No chezmoi-configured routine job targets the bbugyi200 area-note repos (body, cash,
  dev, job, love).
size: small
decisions:
  builtin_mirrors:
    ask:
      Should the builtin external_issue_mirror and external_pr_mirror jobs also skip the
      area-note repos?
    choices:
      leave:
        Leave the builtin mirror jobs alone; they keep polling the area-note repos every
        15 minutes
      exclude:
        Patch both builtin jobs on athena with the same allowlist; new code projects
        need adding too
    default: leave
    why:
      They are builtin, not chezmoi-configured, and mirroring an empty tracker is cheap
      and harmless
    answer: leave
  tailnet:
    ask: Should bbugyi200/tailnet also be dropped from refresh_docs?
    choices:
      keep:
        Keep tailnet in the allowlist; it has no area note of its own and holds real
        host docs
      drop:
        Drop tailnet too, treating it as part of the dev area (dev.md lists it as a
        project alias)
    default: keep
    why:
      tailnet has no area note, and its README and host notes are real docs worth
      refreshing
    answer: keep
proposed_by: bbugyi200.athena.1c8
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-13 10:52:26
status: wip
---

# Exclude Obsidian area-note repos from chezmoi routine jobs

## Goal

The bbugyi200 repos that mirror Obsidian area notes (`~/bob/<area>.md`,
`type: [[area]]`) were recently registered as SASE projects. They only hold `AGENTS.md`,
`CLAUDE.md`, and `sase/`, but the scheduler fans routine jobs out over **every** enabled
project, so they now get documentation-refresh agents. On 2026-11-13
`chop.refresh_docs.cash.3_733386.1` failed and `.2` sat waiting on it. Stop every
chezmoi-configured routine job from targeting these repos.

## Findings (verified while planning)

Area notes with a matching enabled SASE project (`sase project list` on athena): `body`,
`cash`, `dev`, `job`, `love` (projects `gh_bbugyi200__{body,cash,dev,job,love}`). The
other area notes (`meet`, `gtd`, `family`, …) have no project yet.

The other enabled projects are code repos and stay in scope: `actstat`, `bob-cli`,
`sase`, `sase-spotify`, `tailnet`.

Chezmoi-configured routine jobs (`home/dot_config/sase/sase.yml` and
`home/dot_config/sase/sase_athena.yml`; the apollo and kellys_mbp overlays configure
none):

| Routine.job                 | Targeting                                                | Hits area repos? |
| --------------------------- | -------------------------------------------------------- | ---------------- |
| `telegram.tg_outbound`      | none (notification store)                                | no               |
| `run_every.toobig_split`    | `for_each: {source: projects, names: [sase]}`            | no               |
| `ci_watch.ci_watch`         | explicit `sase-org/*` `repos` + `actstat-ci-watch.yml`   | no               |
| `refresh_docs.refresh_docs` | `for_each: {source: projects}`, so every enabled project | **yes**          |

`sase axe job list` confirms this. It shows `refresh_docs[body]`, `[cash]`, `[dev]`,
`[job]`, and `[love]`, and no other chezmoi job expands to these repos. The builtin
`external_issue_mirror` and `external_pr_mirror` jobs (sase `default_config.yml`) also
fan out to them; see decision `builtin_mirrors`.

The `projects` target source supports only the `name`/`names` and `vcs` filters. There
is no exclude filter (sase-core `crates/sase_core/src/axe_chop/targets.rs`,
`filter_project_rows`). `names` matches the effective project name (`cash`, not
`gh_bbugyi200__cash`).

## Approach

Change `refresh_docs` from "every enabled project" to an explicit **allowlist of code
projects** with `for_each.names`, the filter `toobig_split` already uses.

An allowlist fits this config. `ci_watch` and `actstat-ci-watch.yml` already say new
repos must "never silently widen what the job can act on". It also covers future
area-note repos (`meet`, `gtd`, …) without more edits. The cost is that a new _code_
project needs a one-line addition to get doc refreshes.

Rejected: adding an `exclude`/`exclude_names` filter to the `projects` source. That
needs sase-core Rust, Python schema, and a pin bump plus a release before chezmoi could
use it. It would also make every new area repo opt-out instead of opt-in.

Rejected: disabling the area projects. That would also block launching agents in them,
and `love` has an active claim.

## Changes

Work in the chezmoi repo; open it with `sase repo open chezmoi` and use the printed
path.

1. `home/dot_config/sase/sase_athena.yml`,
   `axe.routines.refresh_docs.jobs.refresh_docs`:
   - Add `names: [actstat, bob-cli, sase, sase-spotify, tailnet]` under the existing
     `for_each: {source: projects}`, sorted alphabetically. Drop `tailnet` if decision
     `tailnet = drop`.
   - Add a short YAML comment above the list. It should say this is a code-project
     allowlist, that bbugyi200 area-note repos (body, cash, dev, job, love, and future
     ones) are excluded on purpose, and that new code projects must be added here.
   - Update the job `description`. The summary becomes something like "Refresh
     documentation for allowlisted code projects after meaningful repository drift"
     (≤100 chars). The body's "for each enabled project" becomes "for each allowlisted
     code project".
   - Update the routine `description` sentence "The current job fans out across enabled
     projects…" to say it fans out across the allowlisted code projects. Keep the
     description grammar from sase
     `docs/reference/automation/scheduler-description-grammar.md`: summary ≤100 chars,
     blank line 2, ≤2000 chars total.

> [!decision] builtin_mirrors = exclude Also add map-form patches under
> `axe.routines.external_mirror.jobs` in `sase_athena.yml` for `external_issue_mirror`
> and `external_pr_mirror`. Each one restates
> `for_each: {source: projects, vcs: [git, gh], names: <same allowlist>}`. Share the
> list with a YAML anchor (`&code_projects` / `*code_projects`) so the three jobs cannot
> drift. Patch only `for_each`. Map-form jobs compose by identity across config layers
> (`docs/reference/automation/scheduler-configuration.md#job-fields`), so the builtin
> description and script carry over. If the merged config rejects a description-less
> patch, copy the builtin descriptions verbatim. Mention in the comment that these
> override builtin jobs.

2. Do not touch `toobig_split`, `ci_watch`, or `tg_outbound`; none of them reach the
   area repos.

## Verification

- `python3 -c 'import sys, yaml; yaml.safe_load(open(sys.argv[1]))' home/dot_config/sase/sase_athena.yml`
  succeeds.
- Run the chezmoi repo's checks (`just check`, or at least `just fmt-check lint`) and
  fix anything the edit causes.
- `chezmoi diff --source <chezmoi checkout> ~/.config/sase/sase_athena.yml` shows only
  the intended hunk.
- Check the expansion end to end. Deploy just this target from the checkout
  (`chezmoi apply --source <chezmoi checkout> ~/.config/sase/sase_athena.yml`), then:
  - `sase axe job list` shows `refresh_docs[...]` only for the allowlisted projects,
    with no `refresh_docs[body|cash|dev|job|love]`. With `builtin_mirrors = exclude`,
    the same holds for both `external_*_mirror` jobs.
  - `sase axe job doctor` reports no new config errors.
- Find out whether the running scheduler picks up config changes without a restart. If
  it does not, propose `sase scheduler restart` through `/sase_gate`; never run it
  directly.

## Follow-ups and notes for the final response

- **File a bug task bead** in the sase project through `/sase_new_task` (task type
  `bug`) for the launch failure in the screenshot. It is separate from this exclusion
  and will keep breaking `refresh_docs` for the remaining GitHub projects:
  - `_project_workspace_ref` in sase `src/sase/axe/_config_targets.py` gives gh
    project-source targets a workspace of `gh:<owner>/<repo>`, so `refresh_docs`
    launches `#gh:bbugyi200/cash` (and `#gh:sase-org/sase` for sase).
  - sase-github's `#gh` xprompt (`src/sase_github/xprompts/gh.yml`) declares `gh_ref` as
    `type: word`. sase's `_WORD_RE` (`src/sase/macro/models.py`, `[A-Za-z0-9._-]+`)
    rejects `/`.
  - Observed failure:
    `Workflow '#gh' argument error: Argument 'gh_ref' expects word … got 'bbugyi200/cash'`
    in `chop.refresh_docs.cash.3_733386.1` (2026-11-13). Host-launched agents use the
    `#gh:gh_<owner>__<repo>` form instead.
  - Leave the fix location (emit the project-key form vs. widen `#gh`'s input type) to
    the bug's owner.
- The config change does not clear the existing failed or waiting
  `chop.refresh_docs.cash.*` agents. Tell the user to dismiss them in the TUI.
- Per the chezmoi repo's gotcha, `chezmoi update -a --force` must run after the commit
  lands. If the host finalizer lands it after your turn, say so in the final response so
  the user (or a gate) runs it.
