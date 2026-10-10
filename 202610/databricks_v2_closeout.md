---
tier: tale
size: small
title: Finish the Databricks v2 report and close sase-1jo
goal:
  Bring databricks_nyc_cv_and_role_pitches_v2.md up to the epic's v2 spec, then close
  epic sase-1jo, run symvision, and mark its plan file done.
bead: sase-1jo
proposed_by: bbugyi200.apollo.sase-1jo.land
create_time: 2026-10-10 16:39:36
status: wip
---

- **PARENT:**
  [202610/databricks_followups.md](https://github.com/sase-org/sase--plans/blob/main/202610/databricks_followups.md)
- **BEAD:** sase-1jo

# Finish the Databricks v2 report and close epic sase-1jo

## Context

Epic `sase-1jo` (plan `plan:202610/databricks_followups.md`) is otherwise complete. Read
it with `sase bead read sase-1jo -r "Need the epic scope, decisions, and plan path"`.
Reviewer decisions are final: `eval_pilot = skip` and `base_cv_sase = add`. Do not build
a harness, do not launch agents, and do not edit the CV sources.

The land agent already verified the closed phases and recorded follow-up triage on the
epic. Do not create task beads. Do not redo this triage.

Verified, leave untouched:

- `bbugyi200/CV` commit `c46ba84` (`sase-1jo.1`) rebuilds the base, Batman, Google, and
  LangChain PDFs. Page counts are 3, 2, 2, and 2. Google ends May 2026 in each. The base
  CV has the approved three-bullet SASE entry (Databricks bullets 1, 2, and 6, no
  Omnigent line). The Databricks CV was not in that commit and is still 2 pages. Stale
  Prometheus, Grafana, Gemini CLI, and "33 pipeline" claims are gone.
- Research commit `2fe342a` (`sase-1jo.2`) is the interview-prep pack. It has the six
  design-story cards and the Agent Quality brief, states `eval_pilot = skip`, and does
  not draft a reason for leaving Google. Its 288 lines against an "about 250" target are
  accepted.
- `sase-1jo.3` closed with no code. `tools/agent_eval_pilot/` does not exist.
- Sase commits after the epic started, `5f47d11935` and `2c9f761210`, do not change the
  seven-harness list, the plugin list, or the SQLite telemetry fact. No sase repo edit
  is required.
- `sase bead epic-symbols sase-1jo` listed nothing on 2026-10-10. `sase bead read`
  showed no parent bead.

The gap is
`202610/databricks_nyc_cv_and_role_pitches/databricks_nyc_cv_and_role_pitches_v2.md`
(commit `15020bd`, `sase-1jo.4`). It is 32 lines. It never fetched the Greenhouse API,
has no commit-SHA table, and omits bryanbugyi.com, the exact GitHub and pdflatex steps,
the LinkedIn checklist pointer, the Omnigent pointer, and the launch-post follow-up.

Out of scope: applying, messaging anyone, editing LinkedIn or GitHub, editing
`BryanBugyi_Databricks_CV.tex`, rebuilding PDFs, and any sase code change. Do not run
`just check-full`. Do not run `just check` unless a file tracked in the sase repo
changes, which this tale should not do.

## Steps

1. Open the research sidecar with
   `sase repo open sase--research -r "Finish the sase-1jo v2 remaining-work report"` and
   edit only
   `202610/databricks_nyc_cv_and_role_pitches/databricks_nyc_cv_and_role_pitches_v2.md`
   in the printed path. Read that repo's `AGENTS.md` or README first if the open command
   names one. Keep the file as the single v2 report. Do not add a second report. Target
   80–130 lines. Link to the original report instead of pasting its LinkedIn copy or
   pitches.

2. Re-fetch both Greenhouse jobs and record the check date plus, for each, the id,
   title, `requisition_id`, `location.name`, every `offices[].name`, `updated_at`,
   `application_deadline`, and the posted pay range:
   - `https://boards-api.greenhouse.io/v1/boards/databricks/jobs/8842963002`
   - `https://boards-api.greenhouse.io/v1/boards/databricks/jobs/8379331002`

   A 2026-10-10 fetch found both deadlines null (still listed). `8379331002` (P-1591,
   Sr. Software Engineer- Backend) is New York in both location and office, pay
   $165,300–$219,675, updated 2026-09-21. `8842963002` (P-1567, Staff Software Engineer,
   Agent Quality) has `location.name` "New York City, New York" and `offices[0]` "San
   Francisco, California", pay $200,000–$265,000, updated 2026-09-24. If that split is
   still present, say so in the report and tell Bryan to open the live posting before
   applying. Use the live fetch if it differs. Do not claim both roles are NYC-only
   while the office field disagrees.

3. Re-check `gh api /users/bbugyi200` and write the live bio, company, hireable,
   location, and blog into the report. On 2026-10-10 they were bio
   `{day: "SWE at Google", night: "Batman of the internet"}`, company null, hireable
   null, location `Cranford, NJ`, blog `bryanbugyi.com`.

4. Re-check the site slug. Open `gh:bbugyi200/bryanbugyi.com` with `sase repo open` and
   read the footer LinkedIn slug in `config.toml`. The epic's plan recorded
   `bryan-bugyi-a3650763`, while every CV uses `linkedin.com/in/bryan-bugyi`. If the
   file still mismatches, name the two slugs and the file. Say that an agent cannot
   rebuild or deploy the site (missing Hugo theme checkout, no CI) and that the public
   CV URL `https://github.com/bbugyi200/CV/raw/master/BryanBugyi_CV.pdf` already serves
   the rebuilt base PDF from `c46ba84` on `origin/master`.

5. Re-check Omnigent once:
   `gh issue list --repo omnigent-ai/omnigent --state open --search "runner in:title" --limit 20`
   and the same with `worktree in:title`. On 2026-10-10 every matching issue had an
   assignee. If that is still true, do not name issues. Point at item 5 of the original
   report's "Application plan and timeline" (optional contribution; community Muse
   harness `R7L208/omnigent-muse`). If you find unassigned runner or worktree issues,
   name at most three. Do not open a PR.

6. Rewrite the v2 report so it contains all of the following.
   - A one-line recommendation: prepare the personal details, then apply to both live
     postings on the same day, and re-open the listings first. Say that no application
     or third-party contact has been made.
   - **Postings**, from the live Greenhouse fetch in step 2, including the Agent Quality
     location/office split when it remains.
   - **Done table** with these SHAs. Do not wait for a SHA of this edit:

     | Work                                                         | SHA                                                  | Bead             |
     | ------------------------------------------------------------ | ---------------------------------------------------- | ---------------- |
     | Databricks CV replacement; Google ends May 2026              | `978ebc0`                                            | before this epic |
     | Base, Batman, Google, and LangChain cleanup and PDF rebuilds | `c46ba84`                                            | `sase-1jo.1`     |
     | Interview-prep pack                                          | `2fe342a`                                            | `sase-1jo.2`     |
     | Eval pilot                                                   | no commit; `eval_pilot = skip`                       | `sase-1jo.3`     |
     | v2 report, first draft                                       | `15020bd`                                            | `sase-1jo.4`     |
     | v2 report brought up to this spec                            | this landing turn; the host commit lands after close | `sase-1jo`       |

     Also note page counts 3/2/2/2, TinyTeX packages `ebgaramond`, `fontawesome5`,
     `listings`, `ulem`, and `tocloft`, and that the prep pack does not invent a
     Google-exit reason.

   - **Bryan-only next steps**, in this order. Each item says why an agent cannot do it
     and gives the exact next step. Link, do not paste, the original report sections
     named below.
     1. Databricks CV. The two `% TODO(bryan)` spots are the contact-line location
        comment and the Google Ad Manager bullets, in `BryanBugyi_Databricks_CV.tex`. A
        third comment above Bloomberg is the optional SRE specific. Confirm May 2026 for
        both the Google end and the SASE start. Then, from the CV repo root, run
        `pdflatex -interaction=nonstopmode BryanBugyi_Databricks_CV.tex` twice, delete
        the `.aux`, `.log`, and `.out` files, and confirm `mutool info` still shows 2
        pages. An agent must not invent the bullets or the town.
     2. LinkedIn. There is no API. Point at the original report section "LinkedIn GitHub
        and site" for the checklist, including the referral search. Do not paste that
        checklist.
     3. GitHub. After the live `gh api /users/bbugyi200` values, give these commands and
        say pinning has no API. Pins from the original report: `sase-org/sase`,
        `sase-org/sase-core`, funky, and cookie.

        ```bash
        gh auth refresh -h github.com -s user
        gh api -X PATCH /user \
          -f bio='{day: "building sase (sase.sh)", night: "Batman of the internet"}' \
          -f company='@sase-org' \
          -F hireable=true
        ```

     4. bryanbugyi.com, from step 4.
     5. Apply to both roles on databricks.com (not LinkedIn Easy Apply) on the same day,
        using the matching pitch in the original "## Pitches" section. Recheck each
        listing that day. Ask the recruiter the questions in "Application plan and
        timeline" item 3. When the launch post ships, send that URL as item 2 of the
        same section describes.
     6. Rehearse `databricks_nyc_cv_and_role_pitches_interview_prep.md`. Do not write
        the "why did you leave Google" answer.

   - **Eval pilot, skip.** Carry this deferred spec, and point at the verbatim note on
     `sase-1jo.3`. It is optional and must not delay applying. Use 8–12 coding tasks
     from already-merged history, each frozen at the selected commit's parent, with that
     commit's tests as the oracle, in a declarative manifest. Check that tests pass, the
     change stays in scope, and required artifacts exist. Keep timeouts and failed runs
     in the denominator. Record harness, model, version, and the exact prompt. Repeat a
     subset and report variance, pass@k, and per-task outcomes. Log MLflow traces, and
     use `mlflow.genai.evaluate` only when it makes the result clearer. Localize one
     regression from a trace. Keep `fakey` conformance separate from real-agent
     task-success scores. A future implementation belongs in `tools/agent_eval_pilot/`
     after reading `tools/AGENTS.md`, adds no runtime dependency (`uv run --with mlflow`
     is the optional MLflow path), may run success checks through `sase tool run`, uses
     a stub launcher in unit tests and an explicitly labeled `fakey` plumbing smoke, and
     sends any real launch through `/sase_run` with `/sase_monitor` for the wait. Its
     README gives one real-pilot command and explains the outcome table and pass@k.
   - **Omnigent**, from step 5.

7. Close the epic in this same turn, after the report is saved and before any step that
   needs this edit's own commit SHA, push, or CI result.
   - Run `sase bead epic-symbols sase-1jo`. The 2026-10-10 run listed nothing. If it now
     lists an entry, resolve it (wire it up, privatize it, add a non-test pragma, or
     delete it) or, only when a still-open later bead needs the exemption, re-key that
     Justfile line to that open bead. Do not close while any `--epic-symbol` entry for
     `sase-1jo` remains.
   - Close with a note that states what was verified. Use this text, adjusted only where
     a re-check in steps 2–5 changed a fact:

     ```text
     Verified sase-1jo.1 c46ba84: base_cv_sase=add, Google ends May 2026, page
     counts 3/2/2/2, stale Prometheus/Grafana/Gemini/33-pipeline claims gone,
     Databricks CV untouched. Verified sase-1jo.2 2fe342a: six design-story
     cards, Agent Quality brief, eval_pilot=skip, no Google-exit draft.
     Verified sase-1jo.3 closed with no harness, matching eval_pilot=skip.
     Completed the v2 report against the phase spec in this turn (first draft
     15020bd; 978ebc0 is the pre-epic Databricks CV). Greenhouse re-checked,
     GitHub profile re-checked, bryanbugyi.com slug re-checked, Omnigent
     runner/worktree issues re-checked. No PROPOSED FOLLOW-UP notes. No sase
     integration edit: 5f47d11935 and 2c9f761210 do not change the harness,
     plugin, or telemetry claims. No parent bead.
     ```

     Command: `sase bead close sase-1jo --note "<that note>"`.

   - If close is rejected for leftover `--epic-symbol` entries, fix them and close
     again. If it is rejected because a phase was never completed, finish or reopen that
     phase, or record a deliberate non-done outcome with
     `--force --reason ... --resolution canceled|superseded`. All four phases are
     already closed `done`. Never use `--force` merely to make the close succeed, and
     never use `--force` to advance a successful landing.
   - After a successful close, run `just symvision` from the sase workspace and wait
     until it exits. It confirms the whitelist is clean. If the only failures are
     unrelated to `sase-1jo`, do not expand into those repairs; append a
     `sase bead note sase-1jo` describing them. If a failure is an `sase-1jo`
     epic-symbol that somehow survived the close, fix it and re-run `just symvision`.
   - Set `status: done` in the YAML frontmatter of the epic plan file. Use the PLAN path
     printed by `sase bead read sase-1jo`. The artifact ref is
     `plan:202610/databricks_followups.md`. Do not change any other frontmatter.
   - Re-read `sase bead read sase-1jo -r "Need the parent link"`. This landing review
     found no parent. If there is still no parent, stop. If a parent phase bead is
     present, verify this epic completed that phase's work, close only that phase with
     `sase bead close <parent-id> --note "<what you verified>"`, and leave the
     containing epic to its land agent. If a parent plan bead is present, review its
     landing note, descendants, notes, and linked plan, and close it only when it is
     still fully complete, after retiring its `sase bead epic-symbols <parent-id>`
     entries, running `just symvision`, and marking its plan file done. Stop at the
     first incomplete or ambiguous parent, note the blocker on that parent, and report
     it.

## Verification

- The v2 file is one report of about 80–130 lines and includes the live Greenhouse
  facts, the SHA table, every Bryan-only next step above, the skip spec, and the
  Omnigent outcome.
- `sase bead read sase-1jo` shows closed.
- `sase bead epic-symbols sase-1jo` prints no entries.
- The epic plan frontmatter contains `status: done`.
- No CV file, sase source file, or interview-prep file changed.
