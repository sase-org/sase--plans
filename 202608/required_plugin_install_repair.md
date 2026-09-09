---
tier: epic
status: done
title: Repair the required-plugin install path
goal:
  "`plugins.required` is satisfiable everywhere sase builds — CI, a fresh ephemeral
  workspace, and a workspace with a stale or dangling install — by installing exactly
  what the project declares and verifying it actually imports; and a required plugin's
  bundled config no longer leaks into test runs."
phases:
  - id: install-repair
    title: Make `_setup-required-plugins` install what the project declares, verified
    depends_on: []
    size: medium
    description:
      "install-repair: rewrite `_setup-required-plugins` to honor each
      `plugins.required` PEP 508 spec, resolve an unpublished sase-org plugin from its
      public git repo, verify the distribution imports after install, and fail with an
      actionable message instead of a raw uv resolution error."
  - id: plugin-config-isolation
    title: Stop required-plugin config from leaking into test runs
    depends_on: []
    size: medium
    description:
      "plugin-config-isolation: decide whether plugin-contributed `sase_config` layers
      belong in test runs, implement that decision, and restore
      `test_tribe_panel_display_config_png_snapshot` to a true premise."
  - id: publish-research-artifacts
    title: Publish sase-research-artifacts and retire the git fallback
    depends_on:
      - install-repair
    size: small
    description:
      "publish-research-artifacts: fix the red release-please Publish workflow in
      sase-research-artifacts, cut the first PyPI release, then pin it in
      `plugins.required` and retire the git-repo fallback branch it needed."
proposed_by: bbugyi200.athena.sase-p3.land
parent_bead: sase-p3
bead_id: sase-p3.15
create_time: 2026-09-09 19:51:24
---

- **PROMPT:**
  [prompts/202608/required_plugin_install_repair.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/required_plugin_install_repair.md)
- **PARENT:** [202608/task_bead_types.md](task_bead_types.md)
- **BEAD:**
  [sase-p3.15](https://github.com/sase-org/sase--beads/blob/main/pages/sase-p3/sase-p3.15.md)

## Why

Epic sase-p3 added `plugins.required` and, with it, a `just install` step
(`_setup-required-plugins`, Justfile) that force-installs every required plugin into the
active venv. That step is wrong in three separate ways, and all three are live on master
right now.

**1. Every CI job on master fails at "Install dependencies."** `sase/sase.yml` requires
`sase-research-artifacts`, which has never been published to PyPI — its release-please
`Publish` workflow has failed on every run with
`GitHub Actions is not permitted to create or approve pull requests`, so no tag and no
release exist. In CI there is no linked checkout either, so `install_one` takes the PyPI
branch and uv aborts:

```
[setup] Installing required plugin sase-research-artifacts from PyPI.
  ╰─▶ Because sase-research-artifacts was not found in the package registry
      and you require sase-research-artifacts, we can conclude that your
      requirements are unsatisfiable.
error: Recipe `_setup-required-plugins` failed with exit code 1
```

Verified on CI run 32115186909 (master `1ad14c1d1`): `lint`, `test (3.12)`,
`test (3.13)`, `test (3.14)`, `perf-floors`, `visual-test`, `coverage-contexts`, and
`ace-page-group-isolation` all fail at that one step. The same failure blocks any fresh
ephemeral workspace, whose first mandatory command is `just install`.

**2. `install_one` treats "a dist-info exists" as "installed".**
`uv pip install --no-deps <name>` is a no-op whenever any distribution of that name is
already present, so the recipe cannot repair a bad install and cannot move a stale one
forward. Two consequences are reproducible today:

- A dangling editable install survives. In workspace `sase_12` before this landing,
  `_editable_impl_sase_research_artifacts.pth` pointed at a
  `sase/repos/linked/sase-research-artifacts` directory that did not exist. The
  dist-info still advertised the `research-highlights` file-hook entry point while
  `import sase_research_artifacts` raised `ModuleNotFoundError`, so every single sase
  command printed
  `Skipping invalid file hook sase-research-artifacts@research-highlights ... unknown file-hook provider`
  and `sase validate` exited 1. The recipe printed
  `[setup] Installing required plugin sase-research-artifacts from PyPI.` then
  `Checked 1 package in 0.13ms` and moved on.
- **The external issue mirror is broken in production.** `plugins.required` lists a bare
  `sase-github`, so the recipe leaves whatever version is already installed in place —
  `0.2.4` in this workspace. The `github` task type ships in `0.2.5` (`sase-github`
  commit `a68b879`, released as `0.2.5` and on PyPI). With `0.2.4` installed,
  `entry_points(group="sase_task_types")` is empty, `github` is absent from the live
  catalog, and `_require_github_task_type()`
  (`src/sase/external_mirror/_issue_apply.py:38`) raises
  `required plugin \`sase-github\` is not
  installed`for **every** mirrored issue create.`sase bead task-type list -a` in this
  workspace confirms the catalog holds only the five builtins.

**3. The committed catalog snapshot is missing `github`, so `just check` is red for
anyone who has the version `plugins.required` is supposed to guarantee.** This is D6's
own failure mode, arriving through the door D6 was built to close.
`sase/task_types.json` was generated on a machine holding `sase-github` `0.2.4`, so it
contains only the five builtins. Install `0.2.5` — which is what a correctly-enforced
`plugins.required` gives you — and `just check` fails:

```
init memory --check failed (exit 1)
  run  init memory  update `github` is not in the committed snapshot
       (sase-github 0.2.5 installed); run `sase memory init`
       ~ update  sase/task_types.json  +16
```

Reproduced on clean master `1ad14c1d1` in workspace `sase_12`. The two symptoms are
mutually exclusive and both wrong: with `0.2.4` installed `just check` is green and the
external mirror is broken; with `0.2.5` installed the mirror works and `just check` is
red. Which one an agent gets depends on whether a linked `sase-github` checkout happens
to exist in their workspace, because `install_one` silently prefers it over the index.

**4. A required plugin's bundled config now merges into every test run.**
`sase-research-artifacts` declares a `[sase_config]` entry point whose
`default_config.yml` ships `ace.tribes.research` (icon `∴`, color `#5FD7AF`). Because
the plugin is now force-installed into every venv, that layer reaches every config load.
Verified with an empty `XDG_CONFIG_HOME` and no project:

```
research: {'icon': '∴', 'color': '#5FD7AF', 'description': 'Research swarm agents ...'}
```

That falsifies the explicit premise of
`tests/ace/tui/visual/test_ace_png_snapshots_agents_tribe_panel.py::test_tribe_panel_display_config_png_snapshot`,
which asserts `"∴" not in research_title.plain` and carries the comment "`research` is a
user-owned tribe configured in the operator's own `~/.config/sase/sase.yml`, not in
sase's bundled defaults, and tests run against bundled defaults only — do not 'fix' this
by re-adding a bundled research entry." The test now fails deterministically. It
survived every phase's verification because the visual suite is excluded from
`just test`, `just check`, and `just check-full`.

Worth stating plainly: the sase-p3 plan never asked for this install step. D7 says "This
is not `repos.linked`: a linked checkout of `sase-github` does not install it", and its
enforcement table routes missing plugins to a doctor ERROR, a fail-closed agent path,
and an interactive install gate — never to an unconditional install inside
`just install`. Phase sase-p3.4 added the recipe so this project's own `sase validate`
could pass; that goal is legitimate, but the implementation has to actually work.

## Context and verified current behavior

- `Justfile:181-199` — `_setup-required-plugins`, invoked from `install` and `_setup`,
  so every `just` recipe re-enters it. `install_one <name> <dir>` installs `-e <dir>`
  when `<dir>/pyproject.toml` exists and otherwise runs
  `uv pip install --no-deps <name>`. The name list is hard-coded in the recipe; it does
  not read `plugins.required`.
- `Justfile:45-46` — `sase_research_artifacts_dir` resolves to
  `sase/repos/linked/sase-research-artifacts` when that path exists, else
  `../sase-research-artifacts`. In a fresh workspace neither exists: `sase/sase.yml:253`
  declares the linked repo **without** `auto_clone` (unlike `sase-core` at
  `sase/sase.yml:239`), and the sibling path is a peer workspace directory, not a
  checkout. `sase repo open sase-research-artifacts` materializes it on demand.
- `sase/sase.yml:275-278` — `plugins.required` currently lists two bare names,
  `sase-github` and `sase-research-artifacts`, with no version specifiers, even though
  D7 documents PEP 508 entries such as `sase-research-artifacts>=0.2` and
  `src/sase/plugins/required.py` already parses them with
  `packaging.requirements.Requirement` and reports a `version_mismatch` issue kind.
- `sase-research-artifacts` is public, sits at `0.1.0`, has no git tags, and returns 404
  from PyPI. Its `publish.yml` is a standard release-please pipeline; the failure is
  purely that the release token is unavailable, so release-please cannot open its
  release PR.
- `sase-github` `0.2.5` **is** on PyPI and does register
  `sase_task_types → github = "sase_github.task_types:GITHUB_TASK_TYPES"`
  (`pyproject.toml:35-36`). Nothing in this repo pins it.
- `src/sase/plugins/required.py` is the resolver, the fail-closed helper, and the
  `missing_required_plugin_message` text. `sase doctor -C plugins.required` and
  `sase validate` consume it. `sase doctor -C config.file_hooks` **is** registered
  (`src/sase/doctor/checks_config.py:77`) and runs — an earlier report that this check
  id was unknown does not reproduce.
- `sase plugin install <name>` resolves through the GitHub-org plugin catalog
  (`src/sase/plugins/catalog.py`), which knows each plugin's repository. It reports
  `Latest unknown — not available from the index right now` for
  `sase-research-artifacts`, so the interactive install gate added by sase-p3.11 cannot
  currently install it either.

## Out of scope

- Changing `plugins.required` semantics, the doctor check, the fail-closed helper, or
  the install gate's UX. The declaration is right; only the install path is wrong.
- Backfilling `task_type` onto already-mirrored beads. D8 settled that.
- Anything about the task-type catalog, chips, or corroboration thresholds. Those phases
  are verified complete.

## Phase 1: Make `_setup-required-plugins` install what the project declares, verified

**id:** `install-repair` · **size:** medium · **depends_on:** none

Rewrite the recipe so that a required plugin is installed from a source that exists, at
the version the project declares, and is proven importable before the recipe reports
success.

1. **Read the declaration instead of hard-coding it.** Drive the loop from
   `plugins.required` in `sase/sase.yml` so the recipe cannot drift from the config the
   doctor check enforces. Honor each entry's PEP 508 specifier, so a declared
   `sase-github>=0.2.5` actually upgrades a stale `0.2.4`. A small helper script under
   `tools/` (unit-testable, per the repo's pyscripts lint) is preferable to more bash in
   the Justfile.
2. **Resolve a source that exists**, in this order: a linked or sibling checkout with a
   `pyproject.toml` (editable, `--no-deps`, as today) → PyPI → for a plugin the index
   does not carry, the public sase-org git repository
   (`git+https://github.com/sase-org/<repo>@master`, `--no-deps`). The catalog in
   `src/sase/plugins/catalog.py` already maps plugin name to repository; reuse it rather
   than hard-coding a URL if that is clean, otherwise a declared fallback table is
   acceptable and must be commented with why it exists.
3. **Verify, do not assume.** After each install, import the plugin's top-level module
   in the target interpreter. A dangling `.pth`, a stale version, or a name-satisfied
   no-op must all be detected here and repaired with `--reinstall`/`--refresh`, then
   re-checked.
4. **Fail with an instruction, not a stack trace.** When no source works, exit with a
   message naming the plugin and the exact next command — `sase repo open <name>` when a
   linked checkout is the intended source, otherwise the install command — instead of
   uv's raw `requirements are unsatisfiable` text.
5. **Pin what the project actually depends on, then make the committed snapshot match.**
   Set `sase-github>=0.2.5` in `sase/sase.yml` `plugins.required`, because that is the
   first release carrying the `github` task type the external issue mirror
   hard-requires. Confirm `sase doctor -C plugins.required` reports the version mismatch
   when an older `sase-github` is installed. Then run `sase memory init` so
   `sase/task_types.json` gains the `github` entry it should always have had, and commit
   it — a snapshot generated below the declared floor is the D6 drift this epic exists
   to prevent. `github` is `agent_creatable: false`, so the generated memory note should
   not change; confirm that, and confirm `sase memory init --check` is clean afterward.
6. Consider adding `auto_clone: true` to the `sase-research-artifacts` linked repo entry
   in `sase/sase.yml` so ephemeral workspaces materialize the checkout the way they
   already do for `sase-core`. Decide deliberately and record the reason either way.

**Verify:** `just install` succeeds from a workspace with no linked
`sase-research-artifacts` checkout (simulate by moving it aside) and from one whose
editable `.pth` dangles; after both,
`.venv/bin/python -c "import sase_research_artifacts, sase_github"` succeeds and
`entry_points(group="sase_task_types")` contains `github`. `sase bead task-type list -a`
shows `github`. `.venv/bin/sase validate` is green with no `Skipping invalid file hook`
line on stderr. `just check` green **both** with a linked `sase-github` checkout present
and with it moved aside — the same tree must not be green one way and red the other.
Then push and confirm CI's "Install dependencies" step passes on every job — this phase
is not done until CI gets past install.

## Phase 2: Stop required-plugin config from leaking into test runs

**id:** `plugin-config-isolation` · **size:** medium · **depends_on:** none

Decide, deliberately and in writing, whether a plugin-contributed `sase_config` layer
belongs in this repo's test runs, then implement that decision.

The argument for isolating tests is the epic's own D6 principle: behavior should be a
function of committed configuration, not of whichever plugins happen to be installed on
the current machine. A test suite whose expected output changes because a required
plugin got installed is exactly the failure D6 set out to design away, and it will recur
for every future plugin that ships a `default_config.yml`. The argument against is that
production loads those layers, so isolating tests hides real merged-config behavior.

Recommended resolution: keep plugin config out of the default test fixture — tests
assert bundled defaults — and cover the merge explicitly in targeted tests that install
or fake a plugin config layer, so the behavior stays tested where it is the subject
rather than as ambient noise in unrelated snapshots.

Whichever way it goes:

- If tests are isolated,
  `tests/ace/tui/visual/test_ace_png_snapshots_agents_tribe_panel.py::test_tribe_panel_display_config_png_snapshot`
  passes unchanged and its comment stays true.
- If plugin config is accepted into tests, rewrite that test's premise and comment
  deliberately, re-record its PNG golden, and sweep for every other test that asserts
  against bundled defaults and would silently change when a plugin is installed.

**Verify:**
`SASE_PYTEST_WORKERS=1 just test-visual tests/ace/tui/visual/test_ace_png_snapshots_agents_tribe_panel.py::test_tribe_panel_display_config_png_snapshot`
passes. `just test-visual` shows no regression against the pre-existing master baseline.
`just check` green, and a targeted test proves the chosen merge behavior directly rather
than through a snapshot.

## Phase 3: Publish sase-research-artifacts and retire the git fallback

**id:** `publish-research-artifacts` · **size:** small · **depends_on:**
`install-repair`

`plugins.required` should name distributions a normal user can install. Make that true.

1. In the `sase-research-artifacts` repo (open it with `/sase_repo`), fix the
   release-please `Publish` workflow. Every run fails with
   `GitHub Actions is not permitted to create or approve pull requests`, meaning
   `SASE_RELEASE_TOKEN` is not available to that repo and the `GITHUB_TOKEN` fallback
   cannot open the release PR. This may require a repository or organization setting
   only the owner can change — if so, stop and report exactly what is needed rather than
   working around it.
2. Land the release PR and confirm the wheel reaches PyPI.
3. Back in this repo, pin the released floor in `plugins.required`
   (`sase-research-artifacts>=<released version>`) and remove the git-repo fallback
   branch phase 1 added, if publishing makes it dead code. Keep the import verification
   from phase 1 regardless — that guards against dangling installs, not against a
   missing index entry.

**Verify:** `pip index`/PyPI shows the released version. In a scratch venv with no
checkouts, `uv pip install sase-research-artifacts` succeeds. `just install` succeeds
with the git fallback removed. `sase plugin install sase-research-artifacts --dry-run`
resolves a real version instead of `Latest unknown`. `just check` green and CI green.
