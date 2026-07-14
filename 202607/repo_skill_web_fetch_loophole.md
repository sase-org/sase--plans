---
create_time: 2026-07-14 07:38:56
status: done
prompt: 202607/prompts/repo_skill_web_fetch_loophole.md
tier: tale
---
# Close the web-fetch loophole in the `/sase_repo` repo-access rule

## Problem

A research agent was asked to study how `steveyegge/beads` evolved on GitHub. Instead of opening the repo through
`/sase_repo` (`sase repo open gh:steveyegge/beads -r "..."`), it loaded a web-fetch tool and fetched `github.com` file
URLs directly (README.md, CHANGELOG.md, plus a FAQ URL that 404'd). The memory rule ("agents MUST use your `/sase_repo`
skill first ... any GitHub repo not linked to the current project") was in its context, and the same agent even used
`sase repo open` correctly for the research sidecar — so it knew the skill. The miss was a categorization failure, and
the current instruction text invites it.

## Root cause

1. **Every operative verb in the rule is filesystem-shaped.** The memory blurb says "read or modify _files_", "use the
   _path_ it prints", "do NOT _locate or clone_". The `/sase_repo` skill likewise says "before reading or modifying its
   files" and "use that printed path". Fetching an HTTPS URL neither locates a path nor clones anything, so an agent
   pattern-matches it to "web research", not "reading files in a repository" — even though a GitHub blob/raw URL _is_
   repository file content.
2. **The task framing primed web tools.** The user prompt said "research on how the project has evolved on GitHub",
   which reads as a web-research task; once WebFetch/WebSearch were loaded for the genuinely-web sources (Medium posts,
   third-party reviews), GitHub file URLs went through the same channel.
3. **There is no mechanical backstop.** SASE launches agents with permissions skipped (`--dangerously-skip-permissions`
   in `src/sase/llm_provider/claude.py`) and its governance (commit finalizer, etc.) is prompt-based. Nothing intercepts
   a `github.com` fetch, so the instruction text is the only line of defense — and it has this gap.

Cost of the miss: the open was not recorded in the `sase repo log` audit trail, and the research was strictly worse (no
local `git log`/file tree; one fetch 404'd where a clone could not have).

## Goals

- Make the instruction text unambiguous: retrieving a repo's file content or history **over the web** (github.com
  blob/raw URLs, raw.githubusercontent.com, repo tarballs, GitHub-API/`gh` content reads) counts as reading that repo
  and must go through `/sase_repo`.
- Keep the rule scoped correctly: web tools remain appropriate for content a clone does not contain — blog posts, docs
  sites, GitHub issue/PR discussions, package registries.
- Deploy the fix everywhere the rule is rendered (project memory + provider shims, and the generated skill files for all
  runtimes), uniformly across agent runtimes.

## Non-goals

- No runtime tool hooks or permission deny-rules. Agents run with permissions skipped by design, SASE has no per-runtime
  tool-hook provisioning today, and per-runtime enforcement branching would strain the uniform-runtimes convention. If
  prompt hardening proves insufficient, a follow-up could add a post-hoc transcript audit (flag `github.com`/raw-URL
  fetches in collected agent logs), but that is explicitly out of scope here.
- No change to `sase repo` CLI behavior; `gh:<owner>/<repo>` external opens already do exactly what is needed.

## Design

### Phase 1 — Harden the memory template

Edit `src/sase/main/init_memory/templates/memory-sase.template.md` (the single source for the "Repositories" section
rendered into every context's `memory/sase.md` and provider shims):

- After the existing "When you need to read or modify files..." paragraph, add a short paragraph closing the transport
  loophole. Draft:

  > This rule applies regardless of transport. Fetching a repository's files or history over the web — github.com
  > file/blob/raw URLs, raw.githubusercontent.com, repo tarballs, or GitHub-API/`gh` file-content reads — counts as
  > reading that repo: open it with `/sase_repo` (unlinked GitHub repos open as external repos, e.g.
  > `gh:<owner>/<repo>`) and read the local checkout instead. Web tools remain appropriate only for content a checkout
  > does not contain, such as blog posts, docs sites, and GitHub issue/PR discussions.

- Extend the final reminder line to name the loophole. Draft:

  > IMPORTANT REMINDER: Do NOT locate, clone, or web-fetch another repo's contents any other way than by using
  > `/sase_repo`!

### Phase 2 — Update the `sase_repo` skill source

Edit `src/sase/xprompts/skills/sase_repo.md`:

- **Frontmatter description** (this is the always-visible trigger text in every agent's skill list): extend it so the
  trigger includes the web case, e.g. append "Also required instead of web-fetching a repo's files or history
  (github.com / raw.githubusercontent.com / GitHub API)."
- **Body**: add a short "Researching External GitHub Projects" section with a concrete example:

  ```bash
  sase repo open gh:steveyegge/beads -r "Research how upstream beads evolved"
  ```

  Then read README/CHANGELOG and run `git log` inside the printed path. State the anti-pattern explicitly: do not
  web-fetch github.com or raw.githubusercontent.com file URLs as a substitute — the checkout gives the full tree and
  history, and the open is recorded in the `sase repo log` audit trail. Note what stays web-appropriate (issues, PR
  discussions, blog posts, docs sites).

### Phase 3 — Update tests pinning the old text

- `tests/main/test_init_memory_handler.py` asserts the current template sentences (the "read or modify files" paragraph
  and the "IMPORTANT REMINDER" line) — update expectations to the new text and add an assertion that the rendered memory
  includes the web-fetch clause.
- Check `tests/main/test_init_memory_chezmoi.py`, `tests/main/test_init_skills_sources.py`, and
  `tests/main/test_init_skills_plan.py` for assertions on the affected strings/skill source and update as needed.

### Phase 4 — Regenerate and deploy

- Skills: `sase skill init --force`, then `chezmoi apply` to deploy generated `SKILL.md` files to their managed
  locations (per the generated-skills memory; never edit the generated files directly).
- Memory: `sase memory init` in this project to refresh `memory/sase.md`, `AGENTS.md`, and provider instruction shims
  from the updated template; use `-c`/`-d` first to review the drift. Other contexts (home/chezmoi, other projects) pick
  the change up the next time their memory init runs against the updated sase build — `sase memory init -c` will report
  the drift there.
- Run `just check` (source + tests changed).

Authorization note: generated memory files and shims change only through the managed `sase memory init` /
`sase skill init` flows above, as a mechanical consequence of the template edit. The user explicitly requested this fix
in conversation, and approving this plan is the user's sign-off for that regeneration.

## Verification

- Unit: updated tests in `tests/main/` pass; `just check` clean.
- Rendered output: after `sase memory init -d`, confirm the project `CLAUDE.md`/`memory/sase.md` contain the new
  transport clause and the extended reminder; after `sase skill init --force` + `chezmoi apply`, confirm the deployed
  `sase_repo` skill shows the new description and section.
- Behavioral spot-check: prompt an agent with a "research <some GitHub project>'s evolution" task and confirm it opens
  the repo via `sase repo open gh:...` (visible in `sase repo log`) rather than fetching github.com file URLs.
