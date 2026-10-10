---
tier: epic
title: Databricks NYC application follow-ups
goal: 'Every recommendation in the research report databricks_nyc_cv_and_role_pitches.md
  that an agent can safely finish is done: the public CV repo is free of stale claims
  with all five PDFs rebuilt, the interview-prep material exists, and the eval-pilot
  harness is built. A concise databricks_nyc_cv_and_role_pitches_v2.md report next
  to the original lists exactly what Bryan still has to do himself.

  '
decisions:
  eval_pilot:
    ask: Should this epic build the SASE agent-eval pilot that the report recommends
      before the onsite?
    choices:
      harness: Build harness, task set, and stub/fakey smoke tests; Bryan launches
        real runs later
      harness_and_runs: Also launch the real-agent pilot (about 24-36 agent runs of
        quota) and write results
      skip: Build nothing; the v2 report carries the pilot spec as Bryan's work
    default: harness
    why: Real runs spend model quota and host capacity, and the pilot must not delay
      applying
    answer: skip
  base_cv_sase:
    ask: Should the base CV (the PDF bryanbugyi.com links to) gain a SASE entry for
      May 2026 to present?
    choices:
      add: Add a three-bullet SASE entry reusing the Databricks CV's verified wording
      dates_only: Keep base CV content; fix only tense and rebuild so Google ends
        May 2026
    default: add
    why: The site links this PDF publicly; without SASE it shows an unexplained gap
    answer: add
phases:
- id: cv
  title: CV repo cleanup and PDF rebuilds
  depends_on: []
  size: medium
  description: 'cv: make the base and Batman CVs build without gutils.tex, remove
    stale SASE claims from the older variants, update the base CV, and rebuild and
    verify every changed PDF.'
- id: prep
  title: Interview-prep pack
  depends_on: []
  size: medium
  description: 'prep: write SASE design-story cards and a short Agent Quality topic
    brief into the research sidecar, next to the original report.'
- id: evalpilot
  title: SASE agent-eval pilot harness
  depends_on: []
  size: large
  description: 'evalpilot: plan and build the small task-success eval over real SASE
    runs described in the report, kept separate from fakey conformance, with real
    runs gated by the eval_pilot decision.'
- id: v2
  title: Remaining-work report (v2)
  depends_on:
  - cv
  - prep
  - evalpilot
  size: small
  description: 'v2: re-verify both postings and write the concise v2 report of what
    was done and what only Bryan can still do.'
proposed_by: bbugyi200.apollo.6k
decided_by: reviewer
decided_via: tui
create_time: 2026-10-10 15:45:38
status: wip
bead_id: sase-1jo
---

- **PROMPT:** [prompts/202610/databricks_followups.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/databricks_followups.md)
- **BEAD:** [sase-1jo](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1jo/README.md)

# Plan: Databricks NYC application follow-ups

## Context

The research sidecar (`sase repo open sase--research`) holds
`202610/databricks_nyc_cv_and_role_pitches/databricks_nyc_cv_and_role_pitches.md` (dated
2026-10-10). Read it with
`sase artifact read research:202610/databricks_nyc_cv_and_role_pitches/databricks_nyc_cv_and_role_pitches.md "<reason>"`.
It recommends two NYC roles (Staff SWE Agent Quality `8842963002`; Sr. SWE Backend, AI
Platform `8379331002`), a Databricks CV in `bbugyi200/CV`, LinkedIn/GitHub updates,
pitches, an optional eval pilot, an optional Omnigent contribution, and interview prep.

Facts established while planning (2026-10-10). Phases should re-check them, not trust
them blindly:

- **The urgent item is already done.** `bbugyi200/CV` `origin/master` is `978ebc0`
  ("fix: replace Databricks resume with verified claims and end Google role in May
  2026"). The bad `188f878` CV has been replaced, and all four older `.tex` variants end
  Google in May 2026. Do not touch `BryanBugyi_Databricks_CV.tex`. Its two
  `% TODO(bryan)` spots (Google bullets, location) are Bryan's.
- **Two PDFs are stale.** `BryanBugyi_CV.pdf` (base) still prints "June 2022 - Current",
  and `BryanBugyi_Batman_CV.pdf` still prints "June 2022 – present". The base PDF
  matters most: bryanbugyi.com's "CV" link points at
  `https://github.com/bbugyi200/CV/raw/master/BryanBugyi_CV.pdf`.
- **Why they could not be rebuilt:**
  - Both `.tex` files `\input{/home/bryan/Sync/lib/latex/gutils.tex}`. That file is gone
    from apollo, athena, and the mac. It is also absent from the chezmoi repo and from
    GitHub code search, so treat it as unrecoverable.
  - The Batman CV's preamble comment records what gutils provided: amsmath, amssymb,
    amsthm, bm, enumitem, float, graphicx, listings, ulem, xparse, plus a
    `\declarecommand` macro. Its usage (`\declarecommand{\sub}{m}{...}`) matches
    xparse's `\DeclareDocumentCommand` signature.
  - The TeX install is TinyTeX at `~/.TinyTeX`. It is user-owned, so `tlmgr install`
    needs no sudo. It is missing `ebgaramond`, `fontawesome5`, `listings`, `ulem`, and
    `tocloft`.
- **Stale SASE claims in the older variants:**
  - Affected files: `BryanBugyi_Batman_CV.tex`, `BryanBugyi_Google_CV.tex`, and
    `BryanBugyi_LangChain_CV.tex`.
  - "orchestrates Claude, Gemini, and Codex" (and "Claude Code, Gemini CLI, and Codex").
    SASE now supports seven harnesses: Claude Code, Codex, Antigravity, Qwen Code,
    OpenCode, Muse Code, and Grok Build (`README.md`).
  - "Prometheus telemetry surfacing 33 pipeline metrics". That stack was removed;
    telemetry is a SQLite store behind the Rust core (`docs/telemetry.md`).
  - The plugin list "(Git, GitHub, Chezmoi, Telegram, Gemini, Neovim)" does not match
    `docs/plugins.md`. That file lists sase-github, sase-telegram, sase-nvim,
    sase-listen, and sase-research-artifacts.
  - "Prometheus / Grafana" appears in the Tools rows.
- **GitHub profile.** The `gh` token has scopes `admin:public_key`, `gist`, `read:org`,
  and `repo`, but no `user` scope. An agent therefore cannot PATCH the profile. Current
  values:
  - bio `{day: "SWE at Google", night: "Batman of the internet"}`
  - company null, hireable null, location `Cranford, NJ`, blog `bryanbugyi.com`
- **bryanbugyi.com.** Its source is `bbugyi200/bryanbugyi.com` (Hugo). It is a 2018 blog
  that never mentions Google.
  - Footer LinkedIn slug: `bryan-bugyi-a3650763` in `config.toml`.
  - Every CV uses `linkedin.com/in/bryan-bugyi`, which answers HTTP 200.
  - Its `themes` symlink points to a missing `~/.hugo-themes`, and there is no CI
    deploy, so agents cannot rebuild or deploy it.

Outward-facing effects that approving this plan authorizes:

- The host commits and pushes the `bbugyi200/CV` changes, which is a public repo, as the
  previous research turn did.
- The cv phase runs `tlmgr install` into `~/.TinyTeX`.
- New files go into the public research sidecar.
- Under `eval_pilot = harness_and_runs` only, real agent runs spend model quota.

Out of scope for every phase: anything that impersonates Bryan or contacts a third
party. That covers LinkedIn edits, GitHub profile edits, applications, recruiter or
referral messages, Omnigent PRs, and site deploys. Those go into the v2 report.

## Phase cv: CV repo cleanup and PDF rebuilds

Open the repo with `sase repo open gh:bbugyi200/CV -r "<reason>"` and work only in the
printed path. Before editing, keep the pre-change PDFs (`git show HEAD:<file>.pdf`) in a
scratch dir as the visual baseline.

1. **TeX packages.** Run `tlmgr install ebgaramond fontawesome5 listings ulem tocloft`.
   Then iterate on any further missing `.sty` that `pdflatex` reports
   (`tlmgr search --global --file <name>.sty`). Use no sudo, and do not touch the system
   TeX.
2. **Drop the gutils dependency.** In `BryanBugyi_CV.tex` and
   `BryanBugyi_Batman_CV.tex`, replace the `\input{.../gutils.tex}` line with an
   explicit, commented block. It loads only the packages each file actually needs from
   the list above, and defines `\declarecommand` as xparse's `\DeclareDocumentCommand`
   (for example, `\let\declarecommand\DeclareDocumentCommand`).
   - Load `ulem` with `normalem` unless the baseline render shows underlined `\emph`.
   - The goal is a render that matches the baseline except for intended text changes.
3. **Stale claims (Batman, Google, LangChain variants).**
   - Replace the "Claude, Gemini, and Codex" wording in profiles, project subtitles, and
     skill rows with the seven-harness fact. Short form: "seven coding-agent CLIs
     (Claude Code, Codex, Antigravity, and others)".
   - Replace the Prometheus telemetry clause with the SQLite telemetry fact, or drop it.
   - Correct the plugin list against `docs/plugins.md`.
   - Remove "Prometheus / Grafana" from Tools.
   - Verify the remaining SASE bullets in those files against the sase checkout
     (`README.md`, `docs/`). Make minimal wording changes, and add no claim that the
     Databricks CV does not already make.
   - Afterwards, `grep -n -i -E 'prometheus|grafana|gemini|33 pipeline'` over those
     three files must return nothing stale.
4. **Base CV text.** Fix the Google entry: "Supporting … working working on" becomes
   past tense with the duplicate word removed.

   > [!decision] base_cv_sase = add Add a SASE experience entry above Google using the
   > base CV's own macros.
   >
   > - Heading: "sase — Structured Agentic Software Engineering".
   > - Role: "Creator & Lead Engineer (independent, full-time)".
   > - Dates: "May 2026 - present".
   > - Bullets: three, condensed from Databricks CV SASE bullets 1, 2, and 6 (provider
   >   contract over seven CLIs; ToolRun verification and triage; side project since
   >   2025, on PyPI, sase.sh). Leave out the Omnigent line.
   > - Keep the base CV at its current page count.

   > [!decision] base_cv_sase = dates_only Make no content change beyond the tense fix.

5. **Rebuild.** Run `pdflatex` (twice where cross-references need it) for the base,
   Batman, Google, and LangChain CVs. Remove `.aux`/`.log`/`.out` byproducts, and do not
   rebuild or edit the Databricks CV.
6. **Verify.**
   - Page counts are unchanged, except for one added page if `base_cv_sase = add` truly
     needs it; report this.
   - `mutool draw -F txt` shows Google ending May 2026 in every PDF, with no "Current"
     or "present" on Google and no stale claim.
   - Render each new and baseline page to PNG (`mutool draw -r 70`) and look at them.
     Differences must be limited to the intended edits.
7. Record in the bead notes the files changed, the packages installed, and anything left
   undone. The v2 phase reads these notes.

## Phase prep: Interview-prep pack

Open the research sidecar (`sase repo open sase--research`). Create
`202610/databricks_nyc_cv_and_role_pitches/databricks_nyc_cv_and_role_pitches_interview_prep.md`
beside the original report. Follow the sidecar README conventions: state the question,
the evidence, and a recommendation. Cite evidence by decision slug, doc path, or commit
SHA. Target at most about 250 lines.

1. **Design-story cards.** The report names six SASE stories:
   - single-turn agents
   - host-owned completion
   - triage never changes an exit code
   - the Rust core boundary
   - two-speed CI
   - crash-safe settlement (`62a8b95c72`)

   Read the matching decision records in one batched call. The slugs are
   `single-turn-agents`, `host-owned-completion`,
   `triage-annotates-does-not-change-exit-codes`, `rust-core-required`,
   `ci-two-speed-split`, and `check-full-is-explicit`:

   ```bash
   sase memory read decisions:<slug> ... -r "<why>"
   ```

   Also read the `62a8b95c72` commit. Each card gives:
   - the claim
   - the rejected alternative(s)
   - the cost
   - what would reopen it
   - one concrete evidence pointer
   - a 45-60 second spoken version
   - the likeliest follow-up question, with an honest answer

   Fold in the report's "Claims to be ready to defend" where they apply (no sandboxing,
   adoption, Rust depth, agent-written volume).

2. **Agent Quality topic brief.** Cover trace-based evals, LLM-judge failure modes,
   nondeterminism (repeated trials, pass@k), and turning production traces into
   regression sets. For each, give:
   - a short primer with 1-2 cited sources (web search; MLflow GenAI eval docs are a
     good anchor)
   - how SASE relates, using only capabilities verified in the checkout
   - one question to ask the interviewer

   State plainly that SASE has no task-success evals unless the eval pilot exists.

3. Do not draft the "why did you leave Google" answer. Only Bryan knows the reason.

## Phase evalpilot: SASE agent-eval pilot harness

This phase is `large`, so it plans before implementing. Its spec comes from the report's
"Application plan and timeline" item 4:

- 8-12 coding tasks with frozen start commits and explicit success checks: tests pass,
  the change stays in scope, and required artifacts exist.
- Timeouts and failed runs stay in the denominator.
- Record harness, model, version, and prompt for every run.
- Repeat a subset to show variance (pass@k).
- Log runs as MLflow traces. Use `mlflow.genai.evaluate` scorers only if they make the
  result clearer.
- Keep `fakey` conformance tests separate from real-agent success scores.
- Report per-task outcomes and one regression localized from a trace.

Constraints for the phase's own plan:

- Put the harness in the sase repo under `tools/agent_eval_pilot/` (read
  `tools/AGENTS.md` first). It is dev tooling, not package code, so the Rust core
  boundary does not apply.
- Add no new runtime dependency to `pyproject.toml`. MLflow stays optional, for example
  via `uv run --with mlflow`.
- Draw tasks from real, already-merged history: the parent commit is the start, and the
  commit's own tests are the oracle. Keep a declarative task manifest.
- Consider running success checks through `sase tool run`, so the ToolRun records and
  triage verdicts become the "trace" used to localize a regression.
- Inside a SASE agent, never call `sase run` directly. Unit tests use a stub launcher,
  and the plumbing smoke uses `fakey` explicitly labeled as conformance-only. Any real
  launch goes through `/sase_run` (LaunchApproval).
- Follow the `lint_and_test` reference memory; `just check` runs through
  `sase tool run`.
- The tool's README gives the single command Bryan runs for the real pilot and explains
  how to read the per-task table and pass@k output.

> [!decision] eval_pilot = harness Stop once the harness, task manifest, tests, and
> fakey smoke pass. Record the exact real-run command in the bead notes for the v2
> report.

> [!decision] eval_pilot = harness_and_runs After the harness lands, launch the real
> pilot through `/sase_run`, and wait with `/sase_monitor`, never by blocking. Then
> write the results to
> `202610/databricks_nyc_cv_and_role_pitches/databricks_nyc_cv_and_role_pitches_eval_pilot.md`
> in the research sidecar: per-task outcomes, variance, and one localized regression.

> [!decision] eval_pilot = skip Close the phase with no changes. Note in the bead that
> the v2 report must carry the full pilot spec.

## Phase v2: Remaining-work report

Open the research sidecar and create
`202610/databricks_nyc_cv_and_role_pitches/databricks_nyc_cv_and_role_pitches_v2.md`.
Read the cv, prep, and evalpilot bead notes and commits first. The report must be
concise (target 80-130 lines), and it should link to sections of the original report
instead of re-pasting its LinkedIn copy or pitches.

1. **Re-verify postings.** Fetch
   `https://boards-api.greenhouse.io/v1/boards/databricks/jobs/8842963002` and
   `.../8379331002` and record whether each is live and the check date.
2. **Done.** One short table: what each phase finished, with commit SHAs, plus the
   already-landed CV replacement `978ebc0`.
3. **Still to do (Bryan only).** List these in order. Each item needs why an agent
   couldn't do it and the exact next step:
   - Fill the two `% TODO(bryan)` spots in the Databricks CV (Google bullets, location).
     Optionally add a Bloomberg SRE specific, and confirm the dates. Then rebuild it
     with the exact `pdflatex` command and check it is still two pages.
   - LinkedIn: point to the original report's checklist section. There is no API.
   - GitHub profile: re-check the current values with `gh api /users/bbugyi200`. Then:
     - run `gh auth refresh -h github.com -s user`
     - run the exact
       `gh api -X PATCH /user -f bio=... -f company=@sase-org -F hireable=true` command
     - pin repos in the web UI (there is no API)
   - bryanbugyi.com: the LinkedIn slug mismatch (`config.toml`), and a rebuild/deploy
     from Bryan's machine. Its CV link already serves the rebuilt base PDF.
   - Apply to both roles on the same day with the matching pitches.
   - LinkedIn referral search; the recruiter questions; a follow-up when the launch post
     ships.
   - Eval pilot next step:
     - harness: the real-run command
     - harness_and_runs: link to the results
     - skip: the full spec
   - Optional Omnigent contribution. If `gh issue list` on the Omnigent repo quickly
     shows unassigned runner or worktree issues, name up to three; otherwise keep the
     report's pointer.
   - Rehearse the prep pack; write the "why did you leave Google" answer.
4. Carry forward anything an earlier phase recorded as unfinished.

## Verification

- The cv phase's text-extraction and PNG checks pass, and the bead notes list them.
- The research sidecar contains the interview-prep file and the v2 file. Under
  `harness_and_runs` it also contains the eval results.
- The evalpilot phase passes `just check` when it changes the sase repo.
