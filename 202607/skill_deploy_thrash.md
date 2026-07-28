---
tier: epic
title: Stop sase skill init skill-deployment thrashing
goal: 'Deployed provider skill files are a pure function of one canonical committed
  source revision, a deploy can never move the destination backwards, and the unlanded
  sase_beads template content is reconciled onto master.

  '
phases:
- id: guard
  title: Source-integrity guard for skill deploys
  depends_on: []
  size: medium
  description: 'guard: refuse a chezmoi skill deploy when the invoking workspace has
    uncommitted xprompt template edits or a HEAD that is not an ancestor of the canonical
    branch, add an --allow-dirty escape hatch, and leave read-only and non-chezmoi
    paths untouched.

    '
- id: manifest
  title: Provenance manifest and monotonic overwrite guard
  depends_on:
  - guard
  size: medium
  description: 'manifest: record the source commit and xprompt-set hash in a chezmoi-versioned
    JSON manifest, then refuse deploys whose source revision is a descendant of or
    unrelated to the recorded one, bootstrapping cleanly when the manifest is missing.

    '
- id: serialize
  title: Serialize the deploy and make it attributable
  depends_on:
  - manifest
  size: small
  description: 'serialize: wrap the chezmoi read-compare-write-commit-push sequence
    in an exclusive lock reusing an existing lock helper, and extend the commit trailers
    to record source revision, workspace, and agent on both the direct and deferred
    deploy paths.

    '
- id: reconcile
  title: Reconcile the unlanded sase_beads template onto master
  depends_on: []
  size: medium
  description: 'reconcile: merge the three divergent sase_beads template variants
    into one template on master, verifying every documented bead command against the
    actual CLI before landing, so the guards do not enforce a source of truth that
    is missing content.

    '
- id: converge
  title: Regenerate from reconciled source and confirm convergence
  depends_on:
  - guard
  - manifest
  - serialize
  - reconcile
  size: small
  description: 'converge: deploy once from a clean canonical workspace, confirm a
    re-run is a no-op and that an older workspace is refused rather than reverting,
    reproduce the original ABA against the fixed code, and remove the orphaned dot_gemini
    skills tree.

    '
- id: docs
  title: Document the corrected skill-deploy workflow
  depends_on:
  - converge
  size: small
  description: 'docs: replace the guidance that tells agents to deploy from a dirty
    tree with the commit-then-deploy workflow, after asking the user directly for
    memory-file approval and falling back to CLI help and docs if it is declined.

    '
create_time: 2026-07-27 15:38:50
status: wip
bead_id: sase-ae
---

# Stop `sase skill init` Skill-Deployment Thrashing

## Problem

Generated provider skill files in the chezmoi repo oscillate between older and newer content. Four consecutive
`chore: regenerate skills via sase skill init` commits landed in ~17 minutes, and one of them reverted the two before
it.

Measured blob identity of `home/dot_claude/skills/sase_beads/SKILL.md` in the chezmoi repo:

| chezmoi commit | time  | blob         | lines |
| -------------- | ----- | ------------ | ----- |
| `8edcee29`     | 12:26 | `51e60f591c` | 216   |
| `929b71d5`     | 15:08 | `b56f270047` | 281   |
| `0bdd0a1b`     | 15:17 | `9dd1f4c149` | 295   |
| `18eb0336`     | 15:24 | `51e60f591c` | 216   |
| `57d679e3`     | 15:25 | `f8581cd42b` | 301   |

`51e60f591c` appears twice. That is a literal ABA revert: a later run restored a strictly older rendering, discarding
the `### history`, `### note`, and `### dep` documentation added by the two runs before it.

## Root Cause

`sase skill init` renders from a **per-workspace, mutable, possibly-uncommitted source** and writes to a **single global
destination** with **last-writer-wins semantics and no provenance**. Three factors compound.

### F1 — The source is the live working tree of whichever workspace invokes the command

`load_xprompts_from_internal()` resolves the template directory through importlib resources
(`src/sase/xprompt/loader_sources.py:128-142`):

```python
candidate = Path(str(importlib.resources.files("sase").joinpath("xprompts")))
```

Under the editable install inside workspace `sase_<N>`, that path is `<sase_N>/src/sase/xprompts`. It is the working
tree, not any committed revision, so **uncommitted edits are deployed**.

Confirmed by inspecting all 20 workspaces' copies of `src/sase/xprompts/skills/sase_beads.md`:

| workspace        | lines   | new sections | `src/sase/xprompts/` dirty |
| ---------------- | ------- | ------------ | -------------------------- |
| `sase_15`        | 296     | 3            | yes                        |
| `sase_18`        | 290     | 3            | yes                        |
| 10 others        | 211     | 0            | no                         |
| 8 others (older) | 177–207 | 0            | no                         |

`sase_15` and `sase_18` hold **different uncommitted versions of the same file** (296 vs 290 lines, same section list).
Neither version exists anywhere in sase git history: `git log --since=2026-07-27 -- src/sase/xprompts/skills/` is empty,
and master `465b95a9f` still has the 211-line template with `### dep add` and no `### history` / `### note`.

### F2 — The destination is global and shared by every concurrent workspace

`CHEZMOI_HOME = Path("~/.local/share/chezmoi/home").expanduser()` (`src/sase/config/core.py:59`) is a module constant.
All 20 workspaces — HEADs spanning 07-18 through 07-27 — render into the same files and push the same repo.

### F3 — Overwrite is decided by byte-equality alone; no provenance, no staleness check

`src/sase/main/_init_skills_rendering.py:276-289`:

```python
if not target.path.exists():
    return "create", detail
existing = target.path.read_text(encoding="utf-8")
if existing == target.content:
    return None
return "overwrite", detail
```

A newer deployment is indistinguishable from a stale one. Bare `sase init` silently forces skills
(`src/sase/main/init_onboarding.py:193-194`):

```python
if spec.name == "skills":
    apply_args.force = True
```

The generated file carries no marker at all — `src/sase/xprompts/skills/SKILL.frame.template.md` is 9 lines of
frontmatter plus body, with template variables locked to `{frontmatter, log_skill_use, skill_name, body}`
(`_init_skills_rendering.py:29-31`). The chezmoi commit records only `SASE_TYPE=skills`, with no source revision,
workspace, or agent, so a thrash event is not even attributable after the fact.

### Resulting failure mode

Two independent thrash axes, both observed:

1. **Stale-vs-new.** A workspace on clean master renders the 211-line template and reverts a deployment made from a
   dirty workspace. This is `18eb0336` restoring `51e60f591c`.
2. **Dirty-vs-dirty.** `sase_15` and `sase_18` are concurrent phases of different bead epics editing the same template,
   each unaware of the other. Each deploy clobbers the other's sections.

### F4 — Aggravating: deploy is decoupled from the sase-repo commit

`sase skill init` commits and pushes chezmoi immediately (`src/sase/main/init_skills_handler.py:283-302`,
`src/sase/main/_init_chezmoi_deploy.py:130-266`), independent of whether the source edit is ever committed to the sase
repo. Consequences visible right now:

- chezmoi `HEAD` permanently carries `### history`, `### note`, and `### dep` documentation that exists in **no** sase
  commit. The deployed artifact is unreproducible from any source revision.
- There is no lock. Concurrent agents interleave read-compare-write-commit-push freely.

## Invariant To Restore

> The deployed skill set must be a pure function of one canonical committed source revision, and a deploy must never
> move the destination backwards.

## Scope Notes

- This is host-machine dotfile deployment, not shared domain behavior a web app or editor frontend would need to mirror.
  It stays in Python; no `sase-core` Rust change is required (see the Rust Core Backend Boundary memory).
- The two dirty workspaces (`sase_15`, `sase_18`) hold the only working-tree copies of unlanded template content. The
  `reconcile` phase must not assume they still exist — it falls back to recovering the content from chezmoi `HEAD`,
  which has the superset.

## Phases

### guard — Source-integrity guard: refuse to deploy from a dirty or non-canonical tree

**Depends on:** nothing.

Add a preflight check that runs before any destination write when `use_chezmoi` is true.

Resolve the sase repo root that backs the resolved xprompts package directory (from
`importlib.resources.files("sase").joinpath("xprompts")`, walk up to the git toplevel). Refuse the deploy when either:

- `git status --porcelain -- src/sase/xprompts/` is non-empty (uncommitted template edits), or
- `git merge-base --is-ancestor HEAD origin/<default-branch>` fails (HEAD carries commits not on the canonical branch).

On refusal, exit non-zero with a message naming the offending files or the unmerged commits, and instructing the agent
to land the template change first. Reuse existing git helpers rather than adding new subprocess wrappers.

Add `--allow-dirty` to `add_skills_init_arguments` (`src/sase/main/parser_init.py:19-70`) as the deliberate escape
hatch, and make it explicit in `--help` that it can revert other agents' deployments.

The guard must **not** fire for:

- `--dry-run`, `--check`, `--diff` (read-only paths, `plan_init_skills` at `init_skills_handler.py:305-359`)
- non-chezmoi mode (`use_chezmoi` false — the destination is `~/.claude/skills`, which is workspace-local in effect and
  has no cross-agent contention)
- the CI invocation (`.github/workflows/ci.yml:92` runs `--force` on a clean checkout, which passes the guard naturally)

Tests in `tests/main/test_init_skills_handler.py` and `tests/main/test_init_skills_sources.py`: dirty tree refused,
clean-and-merged tree allowed, `--allow-dirty` overrides, read-only paths unaffected, non-chezmoi mode unaffected.

### manifest — Provenance manifest and monotonic overwrite guard

**Depends on:** `guard`.

Write a manifest alongside the deployment recording what produced it — source commit SHA, the content hash of the
selected xprompt set, and the wall-clock deploy time. Store it as a single JSON file under the chezmoi source root (for
example `<chezmoi_home>/.sase-skills-manifest.json`) so it is versioned with the skills it describes and one read covers
all providers.

Before writing, compare the recorded SHA against the SHA about to be deployed:

- recorded SHA is an **ancestor** of the incoming SHA → proceed (fast-forward)
- recorded SHA **equals** the incoming SHA → proceed (idempotent re-render; content compare still applies)
- recorded SHA is a **descendant** of the incoming SHA → refuse: this run would move the destination backwards
- SHAs are **unrelated** (neither is an ancestor) → refuse: divergent sources

Refusal prints both SHAs, their commit subjects, and the `--force` escape hatch. `--force` proceeds but must still
record the new manifest entry.

Handle the bootstrap case: a missing or unparsable manifest is not an error — treat it as "unknown", proceed, and write
a fresh manifest.

Do not add the marker to individual `SKILL.md` files. The frame template's variable allowlist
(`_init_skills_rendering.py:29-31`) is deliberately narrow, per-file markers would churn every file on every deploy, and
the manifest gives the same guarantee in one place.

Tests in a new `tests/main/test_init_skills_manifest.py`: fast-forward allowed, backwards refused, divergent refused,
identical allowed, missing manifest bootstraps, `--force` overrides and still records.

### serialize — Serialize the deploy and make it attributable

**Depends on:** `manifest`.

Two independent hardening items in the deploy path:

**Locking.** Wrap the read-compare-write-commit-push sequence in `deploy_to_chezmoi`
(`src/sase/main/_init_chezmoi_deploy.py:130-266`) in an exclusive file lock so concurrent agents serialize instead of
interleaving. Reuse an existing lock helper — `src/sase/memory/locks.py`, `src/sase/axe/lock.py`, and
`src/sase/core/agent_artifact_index_lock.py` are the candidates; pick whichever already provides
timeout-with-clear-error semantics rather than writing a fourth. Waiting must print a message and time out with a clear
error, never block forever.

Note explicitly: the lock alone does **not** fix thrashing — a stale writer still wins the lock and still reverts. It
only prevents interleaved partial writes. The `guard` and `manifest` phases are what stop the revert.

**Attribution.** Extend the commit trailers beyond `SASE_TYPE=skills` (applied at `_init_chezmoi_deploy.py:190-197` via
`src/sase/workflows/commit/runtime_tags.py:84-90`) to also record the source revision, the workspace, and the agent, so
the next thrash event is diagnosable from `git log` alone. Follow the existing `SASE_`-prefixed trailer convention
(`runtime_tags.py:26`); do not invent a second format. Confirm the deferred bare-`sase init` path
(`deploy_deferred_chezmoi`, `_init_chezmoi_deploy.py:299-311`) gets the same trailers.

Tests in `tests/main/test_init_skills_deploy.py`, alongside the existing trailer assertion at line 309: trailers present
on both the direct and deferred paths, lock held during deploy, lock timeout produces a clear error.

### reconcile — Reconcile the unlanded `sase_beads` template content onto master

**Depends on:** nothing. Runs in parallel with `guard`, `manifest`, and `serialize`.

The `### history`, `### note`, and `### dep` documentation exists only in chezmoi `HEAD` and in the two dirty
workspaces. Until it lands in `src/sase/xprompts/skills/sase_beads.md` on master, the first clean-workspace
`sase skill init` after the guards ship will legitimately revert chezmoi again — the guards would be enforcing a source
of truth that is missing content.

Reconcile the three variants into one template on master:

- `sase_15`'s copy (296 lines) — `sase bead note` / `sase bead history` / `--lost-notes` (epic `sase-a1.x`)
- `sase_18`'s copy (290 lines) — `sase bead dep list` / `dep tree` / `dep rm` (epic `sase-a3.x`)
- chezmoi `HEAD`'s `home/dot_claude/skills/sase_beads/SKILL.md` (301 lines) — the superset actually deployed

Prefer the chezmoi `HEAD` rendering as the reconciliation base, since it is the union that was last deployed. Strip the
generated frame (frontmatter and the `sase skill use` audit directive) to recover the template body. If the workspaces
still exist, diff their copies against that base to confirm nothing was dropped.

**Verify every documented command against the actual CLI** before landing — `sase bead history --lost-notes --restore`,
`sase bead dep tree --direction`, `sase bead dep rm`, `sase bead note --author`, `close --force --resolution`. The
deployed text was written by agents mid-epic and may describe behavior that shifted before landing. Documentation that
is wrong is worse than documentation that is missing.

Commit the reconciled template to the sase repo. Do **not** run `sase skill init` from a dirty tree as part of this
phase — that is the exact behavior being fixed.

### converge — Regenerate from the reconciled source and confirm convergence

**Depends on:** `guard`, `manifest`, `serialize`, `reconcile`.

With the guards in place and the template landed, run `sase skill init --force` once from a clean workspace at
`origin/master`. Confirm:

- the deployed `SKILL.md` set matches the reconciled template
- a manifest is written recording that revision
- a second run from the same revision is a no-op ("nothing to commit")
- a run from a workspace pinned to an older commit is **refused** by the `manifest` guard rather than reverting

That last check is the regression test for the whole epic — reproduce the original ABA against the fixed code and show
it is now blocked.

Also clean up the orphaned `home/dot_gemini/skills/` tree in the chezmoi repo. No registered provider maps to `.gemini`
any more — `agy` deploys to `.gemini/antigravity-cli` (`src/sase/llm_provider/agy.py:311-312`) — and nothing in the
deploy path removes stale directories. Confirm it is genuinely unreferenced before deleting.

### docs — Document the corrected workflow

**Depends on:** `converge`.

The `sase/memory/generated_skills.md` note currently says:

> Run `sase skill init --force` after changing any skill source file in `src/sase/xprompts/skills/`

That instruction is precisely what produced the thrashing: it tells agents to deploy from a dirty tree. Update it to the
corrected workflow — commit the template change to the sase repo first, then deploy from a clean merged tree; use
`--diff` / `--dry-run` to preview while iterating; `--allow-dirty` and `--force` are deliberate escape hatches that can
revert other agents' work.

**This phase requires explicit user approval before editing `sase/memory/generated_skills.md`.** Per the Memory File
Edits gotcha, approval recorded in a plan file does not count. Ask the user directly with the `/sase_questions` skill.
If approval is granted, run `sase memory init` afterward to regenerate `AGENTS.md` and the provider instruction shims —
that regeneration needs no separate permission.

If the user declines, land the guidance in the `sase skill init --help` text and the CLI docs instead, and report the
memory note as still stale.

## Verification

Every phase runs `just install` then `just check` (per the build_and_run memory — workspaces are ephemeral, so the
install step is not optional). `reconcile` is a template-only change but still touches `src/`, so it runs the full check
too.

The epic is done when the ABA reproduction in `converge` is blocked, `just check` passes, and chezmoi `HEAD` skills are
byte-reproducible from `origin/master` by any workspace.
