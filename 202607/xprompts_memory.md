---
create_time: 2026-07-09 14:52:23
status: done
prompt: 202607/prompts/xprompts_memory.md
tier: tale
---
# Plan: Add `memory/xprompts.md` long-term memory note

## Goal

Add a new **Tier 2 (long-term) memory note** at `memory/xprompts.md` that agents read on demand (via
`/sase_memory_read`) whenever they work with xprompts / prompt directives **or** launch sase agents. It must cover,
densely and accurately:

1. How xprompts are **invoked** in prompts (`#`/`#!` reference syntax, argument grammar, shorthands).
2. How all **prompt directives** (`%`-prefixed) work.
3. How xprompts are **defined** (`.md` single-part vs `.yml` workflows; discovery order; swarms).
4. **VCS xprompt workflows for git and GitHub (`gh`)** — the workspace + rollover/commit block that should be appended
   to an agent's prompt whenever we launch an agent to work on a task for a particular sase project.

Constraint from the user: keep it **short but informative** — every token must earn its place. Target ~800–1000 tokens
(in line with existing long notes like `tui_perf.md` / `pyvision.md`), using dense tables and terse one-liners rather
than prose.

## Why this is safe to add

The user explicitly requested creating this memory file in this conversation, which satisfies the "memory edits require
explicit user permission" rule. Registering it also regenerates the provider shims (see Registration step); this plan
surfaces that so the reviewer approves it up front.

## Source of truth (already verified against the repo)

Content is drawn from the authoritative in-repo docs and xprompt sources, so it stays accurate:

- `docs/xprompt.md` — reference syntax, arguments, shorthand, typed inputs, discovery order, tags, directives
  (`%model/%m`, `%effort/%e`, `%name/%n`, `%wait/%w`, `%hide/%h`, `%auto/%a`, `%repeat/%r`, `%group/%g`, `%alt/%{}`),
  fan-out, `%xprompts_enabled`, command substitution, multi-agent `---` segments, xprompt swarms, aliases
  (`#c`→`#commit`, `#p`→`#propose`).
- `docs/commit_workflows.md` + `src/sase/xprompts/{git,pr,commit,propose,sync,cd}.yml` — the git/gh workspace refs
  (`#gh:`, `#git:`, `#cd:`) and rollover/commit workflows (`#commit`, `#propose`, `#pr`), `SASE_COMMIT_METHOD`, the
  commit finalizer, and the "agent edits files but never commits directly" contract.
- `memory/README.md` — long-note frontmatter schema (`type: long`, `parent: AGENTS.md`, required `description`, optional
  `keywords`) and the `sase memory init` regeneration flow.

## Design of `memory/xprompts.md`

Frontmatter:

```markdown
---
type: long
parent: AGENTS.md
description:
  Read before authoring/invoking xprompts, using `%`/`#` prompt directives, or launching sase agents; includes the
  git/gh VCS workflow block to append when launching an agent on a project task.
keywords: xprompt, directive, %model, %wait, %auto, swarm, #gh, #git, #commit, #pr, #propose, rollover
---
```

Body sections (kept terse; the draft below is the intended content, subject to token trimming):

### 1. Invoking xprompts

- `#name` = inline reference; `#!name` = standalone `.yml` workflow with no `prompt_part` step. Marker must start the
  string or follow whitespace/`([{"'`. `# Heading` (space after `#`) is never matched.
- Args: `#name(a, b)` positional, `#name(k=v)` named (positional first), quote values with commas (`"..."`/`'...'`),
  multi-line via `[[ ... ]]` text blocks.
- Shorthands: `#name:arg`, `#name:a,b`, `` #name:`arg with spaces` ``, `#name+` (=`:true`); line shorthand `#name: text`
  (to blank line) and `#name:: text` (to next directive).
- Namespacing: `#ns/name`; `__`→`/` (`#a__b` = `#a/b`); aliases `#c`→`#commit`, `#p`→`#propose`.
- Composition: fenced code blocks and `%xprompts_enabled:false … :true` regions are passed through verbatim; `$(cmd)`
  runs command substitution; bodies expand recursively.

### 2. Directives (`%` prefix; stripped before the model sees the prompt)

| Directive                | Alias  | Does                                                                                      |
| ------------------------ | ------ | ----------------------------------------------------------------------------------------- |
| `%model:<m>`             | `%m`   | Pick model/provider (`%m:opus`, `%m:codex/gpt-5.5`); `@x` effort suffix (`%m:opus@xhigh`) |
| `%effort:<lvl>`          | `%e`   | `none/minimal/low/medium/high/xhigh/max`                                                  |
| `%name:<n>`              | `%n`   | Name agent; bare `%name` auto-names; `%n(parent, role)` joins a family                    |
| `%wait:<n>`              | `%w`   | Wait for agent(s); `%wait(time=5m)` / `#t:5m` time floor                                  |
| `%repeat:<k>`            | `%r`   | Run prompt k times                                                                        |
| `%auto[:plan/tale/epic]` | `%a`   | Auto-approve the submitted plan                                                           |
| `%group:<tag>`           | `%g`   | User tag; `%hide`/`%h` hides the agent row                                                |
| `%{a \| b}`              | `%alt` | Fan out into one agent per branch                                                         |

Quote model names with spaces/parens: `%m("agy/Gemini 3.5 Flash (High)")` (colon form can't). Fan out models with
`%{%m:opus | %m:sonnet}`.

### 3. Defining xprompts

- `.md` = one `prompt_part`. Frontmatter: `name`, `description`, `input`, `tags`, `snippet`, `skill`, `xprompts`
  (file-local `_`-prefixed helpers). Body is Jinja2 (`{{ arg }}`, `{% %}`) or legacy `{N}`; `@{{ file }}` inlines a
  file.
- Typed inputs: `word/line/text/path/int/bool/float`; no `default` ⇒ required.
- `.yml` = a workflow: `steps` of `prompt_part` / `python` / `bash` / `agent` / `use: shared/...`, with `input`,
  `output`, `environment`, `if`, `repeat`, `finally`, `hidden`, `tags`.
- Discovery (first-wins): `.xprompts/` (CWD) → `xprompts/` → `~/.xprompts/` → `~/xprompts/` →
  `~/.config/sase/xprompts/<project>/` → `sase.yml` `xprompts:` → plugins → package built-ins.
- **Swarm**: an `.md`/prompt body with top-level `---` separators (outside fences) fans out into one agent per segment;
  segments chain via `%wait`.

### 4. VCS workflows — git & gh (append when launching a project-task agent)

Prepend a workspace ref so the agent runs in the right isolated workspace (a bare prompt normalizes to `#git:home`):

| Ref          | Runs in                                                                |
| ------------ | ---------------------------------------------------------------------- |
| `#gh:<ref>`  | GitHub workspace (project name / ChangeSpec / `owner/repo` / `@agent`) |
| `#git:<ref>` | Bare-git workspace                                                     |
| `#cd:<path>` | A directory, no numbered workspace / VCS work                          |

Append a **rollover/commit** workflow to control how the work lands (each sets `SASE_COMMIT_METHOD` and injects "make
the file changes but do NOT create a commit/branch/PR yourself; a finalizer will"):

| XPrompt      | Produces                                                                     |
| ------------ | ---------------------------------------------------------------------------- |
| `#commit`    | Commit on current branch (+ push)                                            |
| `#propose`   | Saved diff, workspace cleaned (park WIP)                                     |
| `#pr:<name>` | New branch + PR via `gh` (status `draft` default; `#pr(name, status=ready)`) |

Also: `#sync` rebases the workspace and auto-resolves conflicts; `#git:<ref>` checks out a ref and shows the diff.

**The rule agents must follow:** never create git commits/branches/PRs yourself. The provider-neutral **commit
finalizer** runs after your work and invokes the `/sase_git_commit` skill → `sase commit` (honoring
`SASE_COMMIT_METHOD`, `SASE_PR_NAME`, `SASE_PR_STATUS`, `SASE_BUG_ID`). Use `gh` for GitHub API/PR reads only when
explicitly needed. `#pr` auto-sets `PARENT` from the current branch's ChangeSpec and suffixes duplicate names `_<N>`.

Typical project-task launch: `#gh:sase %auto #pr:my_change  <task text>`.

## Steps

1. Create `memory/xprompts.md` with the frontmatter + four sections above, trimmed to be dense and short. Match the
   house style of existing long notes (`memory/pyvision.md`, `memory/tui_perf.md`): bold lead-ins, tables, terse
   imperatives; no fluff.
2. Regenerate memory-derived artifacts (**Registration**): run `sase memory init`. This re-derives the Tier 2 listing in
   `AGENTS.md` and the provider shims (`CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`, `QWEN.md`) and refreshes
   `memory/README.md` stats from the new note's `description`. No hand-editing of those generated files.
3. Format markdown: `just fmt` (prettier) so the new file passes `fmt (markdown)`.

## Verification

- `just install` first (ephemeral workspace may have stale deps), then `just check`. The relevant gates:
  `fmt (markdown)` must be clean, and `SASE validation` (`sase validate`) must pass — it fails on memory-init drift, so
  step 2's regeneration is mandatory. (Pre-existing, unrelated failures noted in auto-memory — e.g. sandbox test
  SIGTERM, llm_provider `default_effort` — are not caused by this change.)
- Spot-check `sase memory list` shows `memory/xprompts.md` as an available long note, and
  `sase memory read xprompts.md --reason <r>` returns the content.
- Confirm token weight is reasonable (`memory/README.md` regenerates an approx-tokens line for the new note; keep it
  comparable to other long notes, not an outlier).

## Out of scope / non-goals

- No changes to xprompt behavior, the commit workflows, or any code — this is documentation-only memory.
- Do not document Mercurial (`#hg`) VCS flows beyond noting it is a separate provider; the user asked specifically for
  git and gh.
- Do not add a Tier 1 (`type: short`, always-loaded) note; this is read-on-demand Tier 2 to keep the always-loaded
  context lean.
