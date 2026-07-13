---
create_time: 2026-07-13 14:11:07
status: wip
prompt: 202607/prompts/repo_command_family.md
bead_id: sase-5x
tier: epic
---
# Plan: The `sase repo` Command Family — list · log · open

## Product context

Epic sase-5w gave SASE a crisp taxonomy — **projects**, **repos** (primary / sidecar / linked), **workspaces** — plus
read-only inventories and a minimal `sase repo list`. But the repo _workflows_ still live in the wrong place and lack
visibility:

- **`sase workspace open` is misnamed and error-prone.** Its real job is almost always "give me a repo checkout inside
  my workspace" — the object is a _repo_, not a workspace. Its UX bears the scars: the `-p/--project` flag actually
  takes a _linked repo name_, and agents must pass a positional `<workspace_num>` that they are told to reverse-engineer
  from their own cwd ("check what directory you were started in"). Every agent-instruction shim on the machine carries
  that awkward incantation.
- **There is no durable audit trail of repo opens.** Today an open writes only per-run marker files
  (`opened_linked_workspaces.json`) into the agent's artifacts dir — great for the TUI's SASE-CONTEXT section and the
  commit finalizer, but useless for answering "which agents opened chezmoi last week, and why?". `sase memory read`
  already solved this exact problem with a project-scoped JSONL log and a `sase memory log` dashboard.
- **`sase repo list` is a skeleton.** Plain uncolored `print()` columns, default scope is "all enabled projects" (unlike
  `sase workspace list`, which defaults to the current project), and it only knows whether a repo is cloned at the
  _primary_ checkout — it cannot answer "does workspace #12 have `sase-core` cloned?".

This plan promotes `sase repo` into a complete, beautiful command family:

- `sase repo list` — every primary/sidecar/linked repo, current project by default or all projects on demand, with
  per-workspace clone status.
- `sase repo open` — the successor of `sase workspace open`, with the repo as the positional argument and the workspace
  number _inferred from cwd_.
- `sase repo log` — a `sase memory log`-style audit dashboard over durable repo-open events.

## Command UX (the product spec)

All three subcommands follow `memory/cli_rules.md`: excellent `-h` output with example epilogs, alphabetically sorted
options, a short alias for every public long option, and colored Rich output. Bare `sase repo` keeps delegating to
`sase repo list` via the central default-`list` machinery (no per-command re-implementation).

### 1. `sase repo open <repo> -r "<reason>" [-w N] [-p PROJECT]`

The repo is the object; the workspace is context that SASE figures out itself.

- **`<repo>` (positional)** — a repo name as shown by `sase repo list`: a linked repo (`chezmoi`, `sase-core`), a
  sidecar (`sase--plans`), or the project's primary repo (primary name or project name accepted). Resolution is scoped
  to the host project; an unknown name fails with the list of valid candidates. Exact linked/sidecar match wins over the
  primary-name match; a genuine ambiguity is an error naming both candidates.
- **`-w/--workspace N`** — which numbered workspace hosts the open. **Default: inferred from cwd** via the managed
  checkout marker (`.sase/checkout.json` → `workspace_num`, already present in every managed checkout); a cwd inside the
  primary checkout (no marker) means the primary context (#0). This deletes the biggest footgun in today's agent
  instructions — agents simply run `sase repo open chezmoi -r "<reason>"` from wherever they were started.
- **`-p/--project NAME`** — the host project. Default: inferred from cwd (marker → ProjectSpec scan), exactly like
  `sase workspace list`.
- **`-r/--reason`** — required, non-empty (unchanged contract).

Behavior by repo kind (all reusing the existing, battle-tested machinery in `workspace_handler_list.handle_open_clean` /
`resolve_checkout_path`):

- **linked / sidecar** → materialize + prepare the clone inside the host workspace checkout, exactly as today.
- **primary** → materialize + prepare the numbered workspace itself (the old `sase workspace open <num>` persona, so the
  whole legacy command is subsumed).

Output contract: **stdout carries the prepared path and nothing else** (agents and scripts consume it); any decoration
or warnings go to stderr. On success the open is recorded twice:

1. **Durable audit event** appended to `~/.sase/projects/<host-project>/repo_opens.jsonl` (new; see Architecture).
2. **Per-run artifact markers** (`opened_linked_workspaces.json` + legacy `opened_siblings.json`) exactly as today — the
   TUI SASE-CONTEXT section, tmux modal, linked-deltas view, and commit finalizer all keep working untouched.

### 2. `sase repo list [-a | -p PROJECT] [-w N] [-j]`

Default scope flips to **the current project** (cwd-inferred), matching `sase workspace list`; `-a/--all` shows all
projects (enabled + disabled, again matching `workspace list --all`); `-p/--project` names one explicitly (`-a` + `-p`
conflict, error). Clone status is always evaluated against one _workspace context_: `-w N` explicit, else the
cwd-inferred workspace, else primary #0.

Single-project view (Rich table in a panel; repo kinds reuse the epic's accent-color language — primary/teal,
sidecar/purple, linked/blue; `✓` green, `✗` warning-styled):

```
╭─ Repos · sase · workspace #10 ──────────────────────────────────────────────────╮
│ NAME             KIND     CLONED  WORKSPACES  PATH                              │
│ sase             primary  ✓            38/38  …/workspaces/sase-org/sase/sase_10│
│ sase--plans      sidecar  ✓            31/38  …/sase_10/sase/repos/plans        │
│ sase--research   sidecar  ✓            12/38  …/sase_10/sase/repos/research     │
│ sase-core        linked   ✗            20/38  …/sase_10/sase/repos/linked/…     │
│ sase-github      linked   ✓            26/38  …/sase_10/sase/repos/linked/…     │
╰─────────────────────────────────────────────────────────────────────────────────╯
```

- **CLONED** — does _this_ workspace have the repo (for the primary row: does the checkout exist).
- **WORKSPACES n/m** — in how many of the project's m registered workspaces the repo is cloned: the at-a-glance answer
  to "which workspaces have this repo?" without an unusable 38-column matrix. The full matrix lives in `--json`
  (`clones: [{workspace_num, path, exists}, …]` per repo).
- PATH is the repo's path within the selected workspace context. Description, source, env var, auto-clone, and SDD
  storage stay in `--json` (and the ACE Repos sub-tab detail panel, which already shows them).
- `--all` renders one panel per project, evaluated at primary context, keeping the `WORKSPACES n/m` column; inventory
  issues render as a dim warnings block (stderr for the plain-text/JSON paths, styled in the human view).

### 3. `sase repo log [-r REPO] [-a AGENT] [-w N] [-i ID] [-p PROJECT] [-j]`

The `sase memory log` sibling, over repo-open events for one project (cwd-inferred, `-p` to override):

```
╭─ SASE Repo Open Log ────────────╮
│ Project      sase               │
│ Filters      none               │
│ Open events  124                │
│ Repos        5                  │
│ Agents       38                 │
╰─────────────────────────────────╯
╭─ Repos (5) ──────────────────────────────────────────────────────────────────╮
│ Repo         Kind    Opens  Agents  Last opened       Last agent  Last reason│
│ chezmoi      linked     41      19  2026-07-13 09:14  sase-land   Update …   │
│ sase-core    linked     38      19  2026-07-12 22:41  fable-code  Fix wire…  │
│ …                                                                            │
╰──────────────────────────────────────────────────────────────────────────────╯
```

- Summary + by-repo panels by default; `-r/--repo` and/or `-a/--agent` filters add the agents panel and an events
  drill-down (timestamp, id, agent, repo, workspace #, reason preview); `-w/--workspace` filters by workspace number;
  `-i/--id` shows one event in full (id-prefix matching with ambiguity errors, same as memory log); `-j/--json` emits
  the deterministic payload. Reuse memory log's visual conventions (panel structure, reason previews, border styling) so
  the two audit dashboards read as one system.

### The migration: `sase workspace open` → `sase repo open`

Following the `sase project` enable/disable precedent exactly:

- `sase workspace open` stays registered but **hidden** (`help=argparse.SUPPRESS`, dropped from the group metavar), its
  handler translating the legacy shape (`-p` = linked-repo-or-project, positional num) onto the new implementation, so
  the durable log captures legacy invocations too. It prints one stderr line —
  `Warning: 'sase workspace open' is deprecated; use 'sase repo open'` (the `query_handler/special_cases.py` warning
  pattern) — and behaves identically otherwise. This alias is load-bearing, not a courtesy: generated instruction shims
  in _other_ projects across the machine still say `sase workspace open` until their next `sase memory init`, and they
  must keep working.
- Agent instructions regenerate from the single source template. Exploration confirmed **nothing shells out to the
  command programmatically** — the TUI/finalizer read the marker files via Python APIs — so the blast radius is: parser
  - handler wiring, one instruction template, generated memory/shim copies (repo + chezmoi home), docs, and tests.

New instruction block (in `src/sase/main/init_memory/templates/memory-sase.template.md`), shorter and footgun-free:

```bash
sase repo open <linked_repo> -r "<reason>"
```

Run it from your workspace directory (the workspace number is inferred from where you run it; pass `-w <workspace_num>`
only when running from outside the workspace). Use the printed path as the only linked-repo path. The IMPORTANT-REMINDER
framing ("do NOT locate linked repos any other way") is preserved verbatim with the new command.

## Architecture

### Durable repo-open log (new module, the `memory/read_log.py` twin)

New frontend-neutral module (e.g. `src/sase/repo_open_log.py`, sibling of `repo_inventory.py`):

- `RepoOpenEvent`: `schema_version` (1), `id` (`uuid4().hex[:12]`), UTC ISO `timestamp`, `project` (host project whose
  workspace hosts the open), `repo`, `repo_kind` (primary/sidecar/linked), `workspace_num`, `path` (the printed path),
  `agent_name`, `agent_source`, `artifacts_dir`, `reason`, `cwd`.
- `append_repo_open_event` / `read_repo_open_events` / filter + summarize helpers, mirroring `memory/read_log.py`
  line-for-line where sensible: locked JSONL (`locked_file` + `fcntl`) at `~/.sase/projects/<project>/repo_opens.jsonl`,
  schema-versioned rows, malformed lines skipped on read.
- **Identity is best-effort, not required** — a deliberate divergence from `sase memory read`'s hard
  `require_agent_identity`. Repo open is also a human command (and the deprecated alias must never start failing); when
  `discover_agent_identity()` finds nothing, the event records the login user with `agent_source: "interactive"` instead
  of refusing the open. Logging must also never _break_ an open: append failures degrade to a stderr warning.

### Inventory enrichment (per-workspace clone status)

`src/sase/repo_inventory.py` (the documented Rust-core migration seam) grows a workspace-clone join: for a host
project's registered workspaces (from `workspace_provider` registry/inventory), compute each repo's in-workspace path
via the existing helpers (`sidecar_repo_clone_dir` / `linked_repo_clone_dir`; the primary repo's per-workspace path is
the checkout itself) and stat it. Additive only: `RepoRecord` keeps its current fields so the ACE TUI panes
(`projects_pane.py`, `project_inventory_panes.py`) continue working unchanged; the enrichment is either new optional
fields or a wrapper record, implementer's choice. Cost is ~(repos × workspaces) stat calls — trivial at current scale,
and the module docstrings keep flagging the Rust seam.

### Rust core boundary

No `sase-core` changes. Project discovery stays Rust-owned behind the existing facades; linked-repo config, SDD store
records, and the workspace store — the inputs to everything here — are Python-owned today, which is exactly why
`repo_inventory.py`/`workspace_provider/inventory.py` exist as documented Python adapter seams. The open log mirrors the
memory read log, which is likewise Python-owned. When those inputs migrate into the core, these adapters (and the log
module) are the seam.

### CLI wiring

`parser_repo.py` gains `log` and `open` subparsers (registration order is cosmetic — the central `_sort_subcommand_help`
alphabetizes); `repo_handler.py`'s `_HANDLERS` gains `log`/`open` and its usage string becomes
`Usage: sase repo {list,log,open}`. The workspace group keeps `list/path/cleanup/repair/migrate` untouched; only `open`
is suppressed-and-forwarded. Entry dispatch (`entry.py`) is untouched (both groups already dispatch).

## Phases

Each phase is one agent instance and lands independently (`just check` green, tests included). Every phase that touches
CLI surface must first read `memory/cli_rules.md` via `sase memory read`; phases needing linked repos (chezmoi) use the
command this epic ships (or its legacy alias) to open them. ⚠ Where a phase edits `memory/*.md` or provider shims, the
implementing agent needs explicit user permission **in its launch prompt** — plan approval alone does not grant it (this
plan was requested by Bryan on 2026-07-13, including "update all agent instructions", but per `memory/gotchas.md` that
authorization does not transfer through plan files).

### Phase 1 — `sase repo open` + durable open log + legacy alias

**Goal:** the new command exists, is fully audited, and the old spelling forwards.

- New `src/sase/repo_open_log.py` (event dataclass, locked append/read, filters, by-repo/by-agent summaries, best-effort
  identity with the interactive fallback) with unit tests over tmp log files, mirroring the `memory/read_log.py` test
  approach.
- `sase repo open` parser + handler: repo-name resolution against the host project's inventory (linked/sidecar/primary,
  ambiguity + unknown-name errors), cwd workspace inference from the checkout marker with `-w` override, `-p` host
  override, required `-r`; delegates to the existing open/prepare machinery (`handle_open_clean` path); prints only the
  path on stdout; appends the durable event and writes the per-run markers exactly as today.
- `sase workspace open` becomes a suppressed forwarding alias (`help=argparse.SUPPRESS`, metavar updated, stderr
  deprecation warning, legacy arg-shape translation onto the shared implementation — including the legacy `-p <linked>`
  meaning and the `-P`/`-c` compat flags). Legacy invocations produce the same durable events.
- Tests: parser help (`test_parser_command_help.py` — new repo-open help + reason-required cases, legacy suppression),
  handler tests for name resolution / inference / forwarding (extend `test_workspace_handler_parser.py` +
  `test_workspace_handler_list_path.py` patterns), marker-compat regression (open still feeds
  `opened_linked_repo_records`).

**Acceptance:** from a numbered workspace, `sase repo open <linked> -r "x"` prints the same path today's
`sase workspace open -p <linked> -r "x" <num>` prints, with no positional number; both spellings append identical events
to `repo_opens.jsonl`; the legacy spelling still passes its existing tests plus emits the warning; markers and their
TUI/finalizer consumers are bit-for-bit unaffected.

### Phase 2 — `sase repo list` redesign

**Goal:** the list answers "what repos, and where are they cloned?" beautifully.

- Inventory enrichment in `repo_inventory.py`: per-workspace clone records for a host project (registry join + existence
  stats), additive to `RepoRecord`; unit tests over fixture projects with multiple workspaces, missing clones, and
  disabled projects.
- CLI behavior: current-project default scope (cwd inference), `-a/--all` (all projects incl. disabled, conflict with
  `-p`), `-w/--workspace` context with cwd-inferred default, `-j/--json` with the `clones` matrix.
- Rich rendering per the mockup (kind accent colors, ✓/✗ clone glyphs, `WORKSPACES n/m` column, per-project panels for
  `--all`, styled issues block), replacing the plain `print()` table; help text + example epilog per CLI rules.
- Docs touch-up limited to `sase repo list` sections (full doc sweep is Phase 4); confirm ACE TUI panes are unaffected.

**Acceptance:** `sase repo list` from this workspace shows the sase project's repos with correct clone status for
workspace #10 and truthful n/m counts; `--all` matches the previous all-projects coverage; JSON is deterministic and
carries the full per-workspace matrix; TUI Repos sub-tab behavior is unchanged.

### Phase 3 — `sase repo log`

**Goal:** the audit trail is inspectable and gorgeous.

- New renderer module (e.g. `src/sase/repo_open_cli_log.py` or alongside the log module) mirroring `memory/cli_log.py`:
  summary panel, by-repo panel, agents panel + events drill-down under filters, `--id` detail view with prefix matching,
  deterministic `--json` payloads.
- Parser: `-r/--repo`, `-a/--agent`, `-w/--workspace`, `-i/--id`, `-p/--project`, `-j/--json`, example epilog.
- Tests mirroring the memory-log suite (rendering snapshots via Rich console capture, filter/id-lookup edge cases, JSON
  determinism, empty-log states).

**Acceptance:** after a few `sase repo open` runs, `sase repo log` summarizes them; every filter combination and the id
drill-down work; `--json` is stable; the dashboard visually rhymes with `sase memory log`.

### Phase 4 — Instruction + documentation migration

**Goal:** every instruction and doc teaches the new command; nothing on the machine still _needs_ the old one.

- Template: rewrite the linked-repos block in `src/sase/main/init_memory/templates/memory-sase.template.md` to the new
  `sase repo open <linked_repo> -r "<reason>"` form (workspace inferred; `-w` escape hatch; reminder framing kept).
- Regenerate via `sase memory init`: repo copies (`memory/sase.md`, `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`, `QWEN.md`,
  `OPENCODE.md`) **and** the chezmoi-managed home copies (open the `chezmoi` linked repo with the new command —
  dogfooding). ⚠ Memory/shim edits: explicit user permission required in this agent's launch prompt.
- Update template-string tests (`test_init_memory_handler.py` incl. the `*_workspace_open` test names,
  `test_init_memory_chezmoi.py`).
- Docs sweep for `workspace open` → `repo open` (+ the new `list`/`log` surfaces): `docs/cli.md` (command table + new
  rows for `repo open`/`repo log`), `docs/workspace.md`, `docs/configuration.md` (flag tables + generated-memory
  guidance), `docs/project_spec.md`, `docs/sdd.md`, `docs/init.md`, `docs/ace.md`, `docs/commit_workflows.md`, root
  `README.md`. The prose mention in `memory/symvision.md` is updated under the same permission callout. Blog posts and
  `CHANGELOG.md` stay untouched (historical).
- Docstring polish where the old command name appears in code comments (`ace/tui/opened_workspaces.py`,
  `agent_workspace_tmux_modal.py`, `workspace_handler_list.py` error string).

**Acceptance:** repo-wide grep for `workspace open` hits only the suppressed-alias implementation/tests, historical
blog/CHANGELOG content, and the deprecation warning itself; `sase validate` memory-freshness gate is green against both
the repo and the chezmoi home copies; a fresh agent following the regenerated instructions can open a linked repo with
zero knowledge of workspace numbers.

## Risks & open questions

- **`sase repo list` default-scope change** (all-enabled → current project) alters a command that shipped days ago.
  Judged safe (the TUI consumes the Python API, not the CLI) and it aligns with `sase workspace list` — flagging for a
  yes/no at plan review.
- **Best-effort identity for the open log** (vs `sase memory read`'s hard requirement) trades a little audit rigor for
  never breaking human/TUI/legacy invocations. The `agent_source: "interactive"` rows keep such opens distinguishable.
  Flagging the divergence for review.
- **Workspace inference edge cases:** cwd outside any checkout (no marker, no ProjectSpec match) keeps today's failure
  mode — a clear error asking for `-p`/`-w`. Inference never guesses.
- **Stale instructions elsewhere on the machine:** other projects' generated shims keep saying `sase workspace open`
  until their next `sase memory init`; the suppressed alias covers them indefinitely. Actually _removing_ the alias is
  deliberately out of scope (future cleanup once regeneration has swept every project).
- **Log growth:** `repo_opens.jsonl` is append-only like `memory_reads.jsonl`, which has no rotation today; volumes are
  comparable (a handful of rows per agent run). Rotation, if ever needed, applies to both and is out of scope.
