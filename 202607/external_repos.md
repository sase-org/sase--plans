---
create_time: 2026-07-13 16:54:57
status: wip
prompt: 202607/prompts/external_repos.md
tier: epic
---
# Plan: External Repos — `sase repo open` for Any Repo, Plus `/sase_repo` + `/sase_project` Skills

## Product context

Epic sase-5x made `sase repo open` the single audited door to every repo SASE already knows: the primary, the SDD
sidecars, and configured linked repos. But agents regularly need a repo SASE has _never heard of_:

- The user says "also fix this in my `dotdrop` project" — a different SASE project, not a linked repo of the current
  one. Today the agent has no sanctioned path and improvises (cd-ing into other checkouts, cloning into random dirs),
  bypassing the audit log, the commit finalizer, and every TUI surface.
- The user says "port the fix from `pallets/click`" — a GitHub repo with no SASE project at all. Same story.
- Declaring one-off repos as `linked_repos` config is the wrong tool: config is durable project wiring, not a per-task
  grant.

This plan introduces a fourth repo kind, **external** — a repo the agent didn't know about before the request — and
makes `sase repo open` accept it: any other SASE project's repo, or (today) any GitHub `owner/repo`, cloned on demand
into the workspace-local `sase/repos/external/` tree, fully audited, and first-class everywhere a linked repo is
first-class: commit finalizer, `sase commit` tracking, live file deltas, per-repo commit diffs, revert,
`sase repo list`/`log`, and the ACE TUI.

The VCS story stays agnostic: GitHub is the only external _provider_ today, but the design routes cloning through the
existing workspace-provider plugin layer (the same pluggy seam that gives `#gh:` its meaning), so a future sase-gitlab
plugin adds `gl:owner/repo` without touching core.

Finally, two new generated agent skills put a friendly face on the whole command family:

- **`/sase_repo`** becomes _the_ way agents learn to work across repos. Generated agent instructions stop embedding the
  raw `sase repo open` incantation and instead point at the skill, with one bright-line rule: touching files in any repo
  other than your own workspace checkout — linked repo, another SASE project, or an unlinked GitHub repo — goes through
  `/sase_repo`.
- **`/sase_project`** teaches the `sase project` command (list/show/enable/disable/alias, `--json`), enabling patterns
  like "launch one agent per enabled project".

## Command UX (the product spec)

### `sase repo open` learns external refs

The positional `REPO` argument grows from "a name shown by `sase repo list`" to a three-tier resolution, first match
wins, never guessing:

1. **Host-project inventory name** (unchanged): linked repo, sidecar, primary/project name. Existing exact-match and
   ambiguity semantics are untouched — external resolution only engages when tier 1 finds nothing.
2. **Another SASE project's name** (new): any registered project (its `.sase` record resolves a primary workspace dir) →
   open that project's **primary repo as an external repo**, cloned _from the local primary checkout_ into the host
   workspace's external tree. Local-clone means this tier is instant, offline, and VCS-agnostic for free.
3. **External VCS ref** (new): `gh:owner/repo`, or bare `owner/repo` shorthand (a `/` can never appear in tier-1/2
   names, so the shorthand is unambiguous; it defaults to the `gh` scheme while that is the only registered scheme) →
   clone on demand via the provider plugin that owns the scheme. The repo does not need any prior SASE registration —
   this is the "regardless of whether it even exists yet" case.

Unknown names that match no tier keep today's error and gain one hint line naming the external syntax (`gh:owner/repo`).
Everything else about the command is preserved: required `-r/--reason`, `-w`/`-p` inference from cwd, **stdout carries
the prepared path and nothing else**, decoration to stderr.

```bash
sase repo open dotdrop -r "Port the launcher fix the user asked for"        # tier 2
sase repo open gh:pallets/click -r "Study upstream option parsing"          # tier 3
sase repo open pallets/click -r "Study upstream option parsing"             # tier 3 shorthand
```

Behavior notes:

- **Idempotent reopen**: if the destination clone already exists and is a valid repo, print its path (no re-clone, no
  cleaning — a reopen mid-run must never wipe agent work in progress). The open is still recorded (event + marker).
- **Clone failures are atomic**: clone into a temp dir, then atomically move into place (the pattern the GitHub plugin's
  SDD-clone path already uses); a failed clone leaves no partial directory. GitHub failures aggregate the SSH/HTTPS
  attempts and keep the actionable "run 'gh auth login'" flavor of message.
- **Missing provider**: a `gh:` ref with no plugin serving the scheme (sase-github not installed, or too old to
  implement the hook) fails with exit 2 and an install/upgrade hint listing the registered schemes.
- Help text and the example epilog document all three tiers per `memory/cli_rules.md` (alphabetical options, short
  aliases, colored output).

### Where external clones live

```
<workspace checkout>/sase/repos/
├── plans/                      # sidecar (existing)
├── research/                   # sidecar (existing)
├── linked/<name>/              # linked (existing)
└── external/                   # NEW
    ├── gh/<owner>/<repo>/      # tier 3 — scheme-namespaced, mirrors the forge's own layout
    └── projects/<name>/        # tier 2 — other SASE projects
```

The existing `/sase/repos/` git-exclude entry and the launch-time `sase/repos` → trash cleanup already cover the new
subtree with zero changes: externals are naturally untracked and naturally ephemeral in numbered workspaces. (In the
primary #0 context externals persist between runs, like everything else in a primary checkout — see Risks.)

**Canonical display name** (used consistently in `repo list`, `repo log`, markers, finalizer prompts, and every TUI
label): `gh:owner/repo` for scheme refs, the bare project name for project refs. The directory layout is reversible to
the canonical name and vice versa.

### `sase repo list` / `sase repo log`

- `repo list` grows an `external` kind row per opened external repo, discovered by scanning the registered workspaces'
  `sase/repos/external/` trees (so the listing reflects what is actually materialized, exactly like the CLONED/
  `WORKSPACES n/m` semantics of the other kinds). Externals sort after linked. New kind accent color: **amber/orange** —
  the "outside the project boundary" signal, distinct from primary/teal, sidecar/purple, linked/blue. JSON payload
  carries the same record shape (`source: "opened external"`, no auto_clone).
- `repo log` accepts and styles `repo_kind: "external"`; events already carry the canonical name via `repo`, so
  filtering (`-r gh:pallets/click`) and the dashboards work unchanged beyond the kind validation set + styles.

### The two skills

Source templates in `src/sase/xprompts/skills/`, auto-discovered by `sase skill init`, deployed to all runtimes
(`skill: true`). Descriptions are the product surface here — every token in agent context helps or hurts — so the plan
specifies them verbatim (final wordsmithing at implementation is fine if meaning is preserved):

**`sase_repo.md`** (`log_skill_use: false` — `sase repo open -r` is already the audit, same rationale as
`sase_memory_read`):

> description: Work with repos through `sase repo` (list, open, log). MUST be used to open any repo other than your own
> workspace checkout before reading or modifying its files: linked repos, sidecars, other SASE projects, or any GitHub
> repo not linked to the current project (opened on demand as an external repo).

Body (concise): the three-tier `open` grammar with one example per tier; the bright-line MUST rule (different SASE
project named by the user, or any GitHub repo not configured as a linked repo → open as external; never locate repos any
other way); use the printed path as the only path for reads/writes; `-w` only when running outside the workspace;
`repo list` to see what exists, `repo log` for the audit trail.

**`sase_project.md`** (`log_skill_use` default true — project commands are not otherwise audited):

> description: Inspect and manage SASE projects with `sase project` (list, show, enable, disable, alias). Use when you
> need the set of enabled projects, one project's lifecycle record, or machine-readable project data — e.g. to fan out
> one agent per enabled project.

Body (concise): `list` state filters + `--json` shape (name/state/launchable highlights); `show`; enable/disable with
the `--force` live-work guard; aliases; the fan-out pattern (`sase project list --json` → one `/sase_run` launch per
enabled project); pairs with `/sase_repo` for cross-project file work.

### Generated agent instructions

The "Linked Repositories" block in `src/sase/main/init_memory/templates/memory-sase.template.md` becomes a
"Repositories" block that teaches the rule instead of the raw command:

- Keeps the configured linked-repo list (when present).
- Replaces "agents MUST run `sase repo open …`" with "agents MUST use your `/sase_repo` skill", preserving the
  printed-path-is-the-only-path rule and the IMPORTANT-REMINDER framing.
- Adds the external rule _in both template branches_ (it applies even when no linked repos are configured): any repo of
  a **different SASE project** or any **GitHub repo not linked to this project** MUST be opened as an external repo via
  `/sase_repo` — never located or cloned any other way.
- Stays concise: the skill carries the how; the instructions carry the when-you-MUST.

## Architecture

### The VCS-agnostic clone seam (workspace-provider hook)

New pluggy hook in `src/sase/workspace_provider/_hookspec.py` (firstresult, mirroring `ws_resolve_ref`'s scheme-dispatch
style):

- `ws_clone_external_repo(scheme, ref, dest_dir) -> ExternalRepoCloneResult | None` — plugins return `None` for schemes
  they don't own. A small `ExternalRepoCloneResult` wire dataclass (canonical name, dest path, default branch) lives
  beside the other hookspec dataclasses.
- Scheme discovery: a registry helper (sibling of `get_ref_patterns`) asks plugins which external schemes they serve —
  either a tiny collect-all hook or an added optional `WorkflowMetadata` field, implementer's choice; used for
  shorthand-default resolution and for error messages listing registered schemes.
- **sase-github** (linked repo) implements the hook for `gh`: reuse `_clone_gh_repo` (SSH→HTTPS fallback,
  non-interactive git env) targeting a temp dir + atomic move. Crucially this path does **not** create a ProjectSpec or
  a `~/projects/github/...` checkout — external opens are workspace-local and leave the project registry untouched.
- A future sase-gitlab registers `gl` and core never changes: that is the whole VCS-agnosticism contract.

Tier 2 (project refs) doesn't touch the hook at all: it resolves the target project's primary workspace dir via the
existing Rust-backed project-record facade and clones locally with the same `ensure_git_clone_at` machinery linked repos
use.

### Domain model + persistence

- **`RepoKind`** (`src/sase/repo_inventory.py`) gains `"external"`; kind order/styles extended in the repo-handler and
  repo-log renderers. Inventory assembly grows an external scan over registered workspace checkouts (reversible
  dir→canonical-name mapping); additive to `RepoRecord`, so the ACE Repos pane and JSON consumers keep working.
- **Path helpers** (`src/sase/linked_repos.py`): `EXTERNAL_REPO_CLONES_SUBDIR = ("sase", "repos", "external")` +
  `external_repo_clone_dir(host_checkout, scheme_or_projects, *parts)` beside the existing linked/sidecar helpers.
- **Opened markers** (`src/sase/_linked_repo_markers.py`): schema v2 → v3; records gain `kind: "linked" | "external"`
  (absent → `"linked"` on read, so v2 files parse forever) and, for externals, the canonical ref. Readers expose kind so
  every downstream consumer (finalizer, TUI SASE-CONTEXT, linked-deltas, revert) can distinguish without new files. This
  marker is deliberately the _single_ live channel for "this run opened repo X" — externals are opened mid-run, so
  launch-time `agent_meta.json.linked_repos` is intentionally not involved.
- **Repo-open log** (`src/sase/repo_open_log.py`): `repo_kind` validation set gains `"external"`. Event shape otherwise
  unchanged — the audit story for externals is: every materialization is an explicit, reasoned, durable event; nothing
  external ever appears without one.

### Commit finalizer + commit tracking

- `DirtyRepo.kind` literal gains `"external"` (`commit_finalizer_types.py`); `collect_dirty_state` picks up external
  candidates from the v3 markers alongside sibling candidates; the prompt labels them ``external repo `gh:owner/repo` ``
  (`commit_finalizer_prompting.py`), with the same cd-then-`/sase_git_commit` instruction and the same
  `SASE_COMMIT_METHOD` propagation as linked repos — **uniformity, no special cases**. The commit workflow itself needs
  nothing: it is cwd-based and `get_vcs_provider(cwd)` already classifies a GitHub clone by its remote.
- Commit tracking (`workflows/commit/commit_tracking.py`): repo attribution for external cwds (match under the external
  tree → canonical name) so `commit_results.json` markers, per-repo diff artifacts, and the primary-diff-path guard all
  behave exactly as they do for sidecars/linked repos today.

### ACE TUI

All consumers key off the markers and the commit markers, so the integration is labeling + grouping, not new plumbing:

- **Live deltas** (`file_panel/_linked_deltas.py`): candidate enumeration already reads opened markers — carry kind
  through `LinkedDeltaGroup`, render external groups with their own glyph + amber accent (criterion: distinguishable
  from `▣` linked at a glance; suggestion `◈`), banner suffix `· external repo`. Slot ids must tolerate `:`/`/` in
  canonical names.
- **Per-commit diffs** (`prompt_panel/_agent_commits.py`): cwd→repo-name attribution for external clone dirs; groups and
  the commit-view modal header show the canonical name (non-primary prefix glyph rules extended to externals).
- **Deltas prompt-panel** (`deltas_builder.py`): external groups under their glyph+name header, after linked groups.
- **Revert** ("Revert agent + linked repos"): external repos recorded in the run's markers join the revert candidate
  set; for externals materialized from a remote, revert means discarding local changes in the clone (never re-cloning
  from the network). Label copy: "Revert agent + opened repos".
- **Admin-center Repos pane**: external rows inherit the inventory change; add the kind style.
- PNG visual snapshots extend the existing linked-repos snapshot suite with an external-repo fixture.

### Rust core boundary

No `sase-core` changes. The repo subsystem is the documented Python migration seam (`repo_inventory.py` module
docstring); externals extend that seam. The only Rust interaction is read-only project discovery (tier-2 resolution via
the existing `list_project_records`/lifecycle facades). When repo inventory migrates into the core, external records
migrate with it.

## Phases

Five phases; each is one agent instance and lands independently (`just check` green, tests included). Dependencies: 1 →
2 → {3, 4} → 5; phases 3 and 4 are parallelizable after 2. Every phase touching CLI surface first reads
`memory/cli_rules.md` via `sase memory read`; phase 4 reads `memory/tui_perf.md`; phases editing the sase-github linked
repo open it via `/sase_repo`'s own subject, `sase repo open sase-github -r "…"`. ⚠ Where a phase regenerates
`memory/*.md`, `AGENTS.md`, or provider shims, the implementing agent needs explicit user permission **in its launch
prompt** — plan approval alone does not grant it (per `memory/gotchas.md`; this plan was requested by Bryan on
2026-07-13 including the instruction updates, but that authorization does not transfer through plan files).

### Phase 1 — External repo domain model + provider seam

**Goal:** the `external` kind exists everywhere data is modeled; nothing user-visible changes yet.

- External ref grammar: parse/canonicalize `gh:owner/repo` + bare `owner/repo` shorthand + reversible dir↔name mapping
  (a small pure module, sibling of `repo_inventory.py`).
- `EXTERNAL_REPO_CLONES_SUBDIR` + clone-dir helpers in `linked_repos.py`.
- `RepoKind` + kind order/styles: inventory scan of workspace `sase/repos/external/` trees (fixture-based unit tests);
  `repo list` renders external rows (amber accent) and `--json` carries them; repo-open-log `repo_kind` validation +
  `repo log` styles.
- Marker schema v3: `kind` field with backward-compatible reads (v2 fixtures), external ref persistence, reader surface
  exposing kind.
- Workspace-provider hookspec: `ws_clone_external_repo` + `ExternalRepoCloneResult` + scheme discovery, with
  hookspec-conformance tests (extend `test_workspace_provider_hookspec.py`).

**Acceptance:** unit suites cover ref parsing round-trips, marker v2→v3 compat, inventory external scan, and the new
hookspec; `sase repo list` against a fixture workspace with a hand-planted external clone shows the amber external row;
no behavior change for existing kinds.

### Phase 2 — `sase repo open` opens external repos end to end

**Goal:** the three-tier resolution ships; opening any repo is one audited command.

- Handler resolution tiers in `repo_handler._handle_open`: inventory (unchanged) → project registry → scheme ref;
  unknown-name error gains the syntax hint; help/epilog rewritten per CLI rules.
- Tier 2: resolve the target project's primary dir (lifecycle facade), local clone into `external/projects/<name>`,
  idempotent reuse.
- Tier 3: scheme dispatch through the new hook; temp-clone + atomic move contract enforced core-side; missing-scheme and
  clone-failure error paths (exit 2, stderr only, no partial dirs).
- **sase-github linked repo**: implement `ws_clone_external_repo` for `gh` (reuse `_clone_gh_repo`; no ProjectSpec, no
  global checkout), plus plugin-side tests. Open the plugin repo via `sase repo open sase-github -r "…"`.
- Recording: v3 markers (kind external, canonical name) + durable open events (`repo_kind: "external"`); stdout purity
  preserved (path only).
- Tests: handler tiers (unknown/ambiguous/idempotent/failure atomicity), marker + log recording, parser help snapshot.

**Acceptance:** from a numbered workspace, `sase repo open gh:<owner>/<repo> -r "x"` prints a path under
`sase/repos/external/gh/…`, reopening prints the same path without re-cloning, the event appears in `sase repo log` with
kind external, and `sase repo list` shows the row; `sase repo open <other-project> -r "x"` does the same from the local
primary without touching the network; all existing repo-open tests stay green.

### Phase 3 — Commit finalizer + commit attribution for externals

**Goal:** an agent that dirties an external repo cannot finish without committing it, and every commit is attributed.

- `DirtyRepo.kind` + `"external"`; `collect_dirty_state` external candidates from v3 markers; prompting labels + the
  uniform cd-then-`/sase_git_commit` + `SASE_COMMIT_METHOD` instruction (extend `test_commit_finalizer_siblings*.py`
  patterns with external cases, incl. marker-compat regression).
- Commit tracking: external cwd→canonical-name attribution (`commit_results.json` markers, per-repo diff artifacts,
  primary-diff-path guard), mirroring the sidecar matcher.

**Acceptance:** finalizer tests prove a dirty external repo blocks completion with a correctly-labeled prompt and a
clean one doesn't; a commit made inside an external clone lands in `commit_results.json` with the canonical repo name
and its own diff artifact, and never clobbers the primary fallback diff.

### Phase 4 — ACE TUI: deltas, diffs, revert, inventory

**Goal:** external repos look native — intuitive, reliable, beautiful — in every Agents-tab surface.

- Read `memory/tui_perf.md` first. Live delta groups (kind-aware `LinkedDeltaGroup`, external glyph + amber accent,
  banner `· external repo`, slot-id robustness for `:`/`/`), deltas-builder grouping, file-panel source labels.
- Commit-diff grouping + commit-view modal header attribution for external repos.
- Revert integration: externals from markers join revert candidates (discard-local-changes semantics, no network);
  modal/help copy "Revert agent + opened repos".
- Admin-center Repos pane external styling (inherits Phase 1 inventory).
- Tests: extend `test_linked_deltas.py`, `test_agent_deltas.py`, revert suites; PNG snapshots via a new external-repo
  fixture beside `test_ace_png_snapshots_agents_linked_repos.py` (goldens updated with
  `--sase-update-visual-snapshots`).

**Acceptance:** with a fixture agent that opened + dirtied an external repo, the file panel shows an external diff slot
with the new glyph/label, the deltas panel groups its entries under the canonical name, commit diffs attribute to it,
revert offers and cleans it, and the visual snapshot suite passes locally with exact pixel equality.

### Phase 5 — Skills, agent instructions, docs

**Goal:** agents are taught the new world; nothing still teaches the raw incantation.

- New skill sources `sase_repo.md` + `sase_project.md` in `src/sase/xprompts/skills/` per the spec above (descriptions
  verbatim, bodies concise); `sase skill init --force` + `chezmoi apply` to deploy across runtimes.
- Template rewrite: memory-sase "Repositories" block per the spec (skill-first, external MUST rule in both branches);
  regenerate via `sase memory init` for the repo copies (`memory/sase.md`, `AGENTS.md`, shims) **and** the
  chezmoi-managed home copies (open `chezmoi` via `/sase_repo` — dogfooding). ⚠ Explicit user permission required in
  this agent's launch prompt for the memory/shim regeneration.
- Docs sweep: `docs/configuration.md` (external repos beside the linked-repos section), `docs/cli.md` (three-tier
  `repo open`, new kind in `repo list`), any other `sase repo open` guidance; update template-string tests
  (`test_init_memory_handler.py`, `test_init_memory_chezmoi.py`) and skill-generation tests.
- `sase validate` green (memory-freshness gate against repo + chezmoi home).

**Acceptance:** all five runtimes receive both SKILL.md files with the specified descriptions; regenerated instructions
contain the skill-first rule and the external MUST rule; repo-wide grep shows no generated instruction telling agents to
run `sase repo open` directly; `sase validate` passes.

## Risks & open questions

- **Commit-method semantics on externals:** uniformity means a `#pr` launch drives `sase commit --type pr` inside an
  external GitHub clone → a PR against _that_ repo, and the COMMITS entry lands on the host project's ChangeSpec (same
  as linked repos today). Judged correct-by-uniformity (per the no-runtime/no-repo special-casing convention) — but it
  is new power (agents opening PRs on arbitrary repos the user named); flagging for a yes/no at review.
- **Scope of tier 2:** project refs open the _primary_ repo only. Opening another project's linked repos or sidecars
  stays out of scope (open that project's repo, then its `sase.yml` tells you what's linked — or extend the grammar
  later, e.g. `project/repo`).
- **Bare `owner/repo` default scheme:** hardcoded to `gh` while it is the only scheme; when a second provider lands this
  becomes config (`external_repos.default_scheme`) — noted in code, not built now.
- **Primary-context persistence:** externals opened in the primary #0 checkout persist across runs (launch cleanup only
  wipes numbered workspaces' `sase/repos`). Accepted — matches every other primary-checkout artifact; `repo list` keeps
  them visible rather than hidden.
- **Staleness on reuse:** idempotent reopen never fetches. Numbered workspaces are wiped between runs so clones are
  fresh in practice; a `--fetch`/freshness story is deliberately deferred.
- **Auth/network failures:** external `gh:` opens are the first repo-open path that needs network + credentials at open
  time. Mitigated by atomic temp-clone, aggregated SSH/HTTPS error text, and the `gh auth login` hint pattern.
- **Plugin version skew:** an older sase-github without the hook makes `gh:` refs fail with the upgrade hint (pluggy
  returns no result); tier 1/2 opens are unaffected.
- **Security posture:** external opens let agents materialize arbitrary user-named repos. The mitigations are the
  existing ones, applied on purpose: a required human-readable reason, a durable audit event per open, `sase repo log`
  visibility, and no auto-clone path — externals only ever appear through the explicit command.
