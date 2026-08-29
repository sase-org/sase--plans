---
tier: tale
size: small
title: Apply the AGENTS v3
goal:
  The generated Repositories section carries an `IMPORTANT REMINDERS:` bullet list whose
  first bullet allows `sase artifact read` alongside `/sase_repo` and whose second
  bullet makes audited artifact reads mandatory for sidecar artifacts, with the
  superseded "Prefer an audited read" paragraph removed.
proposed_by: bbugyi200.athena.0g0.w0
create_time: 2026-08-29 09:19:03
status: wip
---

# Plan: Apply the AGENTS v3 `#b` annotations to the Repositories memory section

The user annotated a rendered copy of this project's generated agent instruction file
and asked for the two annotations tagged `#b` to be applied. Both land in the same
place: the `## Repositories` section of the generated SASE memory note. This plan
converts the `IMPORTANT REMINDER:` line into an `IMPORTANT REMINDERS:` bullet list and
folds the artifact-read rule into it.

## Context

The canonical source for that section is the Jinja template
`src/sase/main/init_memory/templates/memory-sase.template.md`. Everything else with this
text — `sase/memory/sase.md`, `AGENTS.md`, and the `CLAUDE.md` / `GEMINI.md` / `QWEN.md`
/ `OPENCODE.md` provider shims — is generated from it by `sase memory init` and must
never be hand-edited.

The two annotations were:

1. On the paragraph beginning "Prefer an audited read over opening a repo": replace it
   with a new sentence, and move that sentence into a bullet below the
   `IMPORTANT REMINDER:` line, which is renamed to `IMPORTANT REMINDERS:` and has its
   current contents migrated into the first bullet.
2. On the `IMPORTANT REMINDER:` line itself: mention that `sase artifact read` is also
   allowed, and convert the line into a bullet.

Commit `affc43a6f` ("docs(memory): apply AGENTS v2 #a annotation trims") is the
precedent for this kind of change and shows the expected commit shape: template edit,
regenerated outputs, and test updates all in one commit.

## Steps

### 1. Install the workspace virtualenv

Run `just install` first. It is required before step 3 and before verification, because
an agent workspace clone owns an isolated virtualenv that may be missing the built
`sase_core_rs` extension.

### 2. Edit the template

In `src/sase/main/init_memory/templates/memory-sase.template.md`, replace this block:

```markdown
Prefer an audited read over opening a repo: read memory notes with `sase memory read`,
and always read artifact files stored in sidecar repos with
`sase artifact read <ref> "<reason>"`.

IMPORTANT REMINDER: Do NOT locate, clone, or web-fetch another repo's contents any other
way than by using `/sase_repo`!
```

with this block:

```markdown
IMPORTANT REMINDERS:

- Do NOT locate, clone, or web-fetch another repo's contents any other way than by using
  `/sase_repo` or `sase artifact read`!
- The `sase artifact read <ref> "<reason>"` command MUST be used to read artifacts (so
  the reads are audited) from sidecar repos. Do NOT read sidecar artifact files
  directly.
```

Formatting constraints that matter here, from `src/sase/main/init_memory/formatting.py`:

- Keep the blank line between `IMPORTANT REMINDERS:` and the first bullet. The generated
  files are prettier-checked, and the surrounding template already separates a lead-in
  paragraph from its bullet list this way.
- Keep bullets at column 0. The formatter only recognizes a list item when the line
  starts with `- ` with no leading indentation; it re-wraps each bullet itself and
  supplies the two-space continuation indent, so the exact wrapping written into the
  template does not need to match the generated output.

Do not restore the dropped `sase memory read` clause. Memory reads are already covered
by the SASE Memory section above it, which routes reference-note reads through the
`/sase_memory_read` skill.

### 3. Regenerate the derived files

Run memory init **through the workspace virtualenv**:

```bash
.venv/bin/sase memory init
```

This is not optional pedantry. The `sase` on `PATH` is a `uv tool` editable install that
points at the user's primary sase checkout, not at this workspace, so it renders the
_primary checkout's_ copy of the template. Running the `PATH` binary here would silently
revert already-landed template changes — for example, `sase memory init --check` run
that way currently proposes reverting commit `80f389d74`. Confirm you used the right
binary: after regenerating, `.venv/bin/sase memory init --check` must report no pending
memory work, and `git diff` must show changes only in the Repositories section (plus the
line and token counts in `sase/memory/README.md`).

Expect regenerated changes in `sase/memory/sase.md`, `sase/memory/README.md`,
`AGENTS.md`, `CLAUDE.md`, `GEMINI.md`, `QWEN.md`, and `OPENCODE.md`.

Running the template block above through `format_generated_memory_markdown` produces
exactly this, which is what the regenerated files should contain:

```markdown
IMPORTANT REMINDERS:

- Do NOT locate, clone, or web-fetch another repo's contents any other way than by using
  `/sase_repo` or `sase artifact read`!
- The `sase artifact read <ref> "<reason>"` command MUST be used to read artifacts (so
  the reads are audited) from sidecar repos. Do NOT read sidecar artifact files
  directly.
```

### 4. Update the test assertion

`tests/main/test_init_memory_handler_repositories.py` asserts the old wording at lines
100-102:

```python
        assert (
            "IMPORTANT REMINDER: Do NOT locate, clone, or web-fetch another "
            "repo's contents any other way than by using `/sase_repo`!"
        ) in memory_line
```

Replace it with assertions covering both new bullets, so each annotation is pinned:

```python
        assert (
            "IMPORTANT REMINDERS:"
        ) in memory_line
        assert (
            "- Do NOT locate, clone, or web-fetch another repo's contents any "
            "other way than by using `/sase_repo` or `sase artifact read`!"
        ) in memory_line
        assert (
            "- The `sase artifact read <ref> \"<reason>\"` command MUST be used "
            "to read artifacts (so the reads are audited) from sidecar repos. "
            "Do NOT read sidecar artifact files directly."
        ) in memory_line
```

`single_line` is the helper already imported by that module; it collapses whitespace, so
match against `memory_line` rather than the raw text. This is the only test in the repo
that asserts the changed wording —
`grep -rn "IMPORTANT REMINDER\|Prefer an audited read" tests/` should return nothing
else after the edit.

## Out Of Scope

- `src/sase/xprompts/skills/sase_repo.md` already recommends
  `sase artifact read <ref> "<reason>"` for single-artifact context and never authorizes
  direct sidecar artifact reads, so it stays consistent with the new wording. Leave it
  alone; editing it would drag in the commit-then-deploy chezmoi workflow for generated
  skills for no behavior change.
- `sase/memory/sase_artifacts.md` already says to use an audited read whenever an
  artifact is context for your work. No change needed.
- The `#a`-tagged annotations in the same source document. Annotation `#a` on the memory
  section was already applied by commit `80f389d74`; the user scoped this request to
  `#b`.
- The user's annotated Markdown file itself. It is a generated highlights sidecar with a
  content marker hash, and the equivalent v2 annotations were left in place after being
  applied.

## Verification

```bash
just check
```

Hand `just check` to your `/sase_monitor` skill if it runs long. Nothing here touches
the broadening set, so `just check-full` is not required.
