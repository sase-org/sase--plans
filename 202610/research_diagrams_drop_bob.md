---
tier: tale
title: Replace bob-cli with Mermaid CLI in sase-research-artifacts
goal:
  The research swarm's diagram check and all docs/tests in sase-research-artifacts work
  without bob-cli, using Mermaid's official CLI instead.
size: small
decisions:
  npx_fallback:
    ask:
      With no working mmdc installed, should the check fall back to `npx -y -p
      @mermaid-js/mermaid-cli mmdc`?
    choices:
      allow:
        Works for anyone with Node 22.13+; first run downloads mermaid-cli and Chrome
        (~150 MB)
      installed_only:
        Uses only a pre-installed working mmdc; never downloads, otherwise diagrams fail
    default: allow
    why: Mirrors the audio macro's `uvx sase-listen` fallback, so diagrams need no setup
    answer: allow
proposed_by: bbugyi200.athena.11o
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-19 07:38:23
status: wip
---

# Replace bob-cli with Mermaid CLI in sase-research-artifacts

## Problem

`#research_swarm` runs `#research/diagrams` inside its `<clan>.image` segment whenever
`image=true` or `diagrams=true`. Step 4 of that macro ("Validate") requires every
Mermaid diagram to pass:

```sh
bob ref create <scratch>/check.md -o <scratch>/check_render.pdf -n -P <parent>
```

It also needs a "Bob area or project note" for `-P`, falls back to
`bob capture-targets`, reads bob's `diagrams: rendered N of N Mermaid diagrams` line,
and reports `"bob render check unavailable on this host"` when bob is missing. bob-cli
is Bryan's personal Obsidian/Highlights tool, not a dependency of
`sase-research-artifacts`. So on every machine without it, the diagrams feature always
ends with `status: failed` and never proposes a diagram.

All bob-cli couplings in the linked `sase-research-artifacts` repo (`CHANGELOG.md`
history aside):

- `src/sase_research_artifacts/xprompts/research_diagrams.md`: step 4 (the bob command,
  `-n`/`-P` notes, `bob capture-targets`, the "If bob is missing" rule), step 6 (the
  manifest example `check: "diagrams: rendered 1 of 1 Mermaid diagrams"` plus the bullet
  "`check` is bob's `diagrams:` line"), and step 7 (the `failed` example reason
  `bob render check unavailable on this host`).
- `docs/macros.md`: the `#research/diagrams` manifest example `check:` line, the whole
  `### Validation` subsection, and the `failed` JSON example.
- `README.md` and `docs/configuration.md`: the `research-highlights` wiring example
  `command: bob highlights create --include-id`, plus the rationale sentences "Bob
  derives its PDF basename and marker id ..." and "... leak into Bob's derived PDF
  basename and marker id".
- `docs/architecture.md`: "the basename-sensitive Bob Highlights command".
- `tests/test_macro_loading.py::test_research_diagrams_restraint_check_and_anchor_rules`
  asserts the bob command. `tests/test_provider_specs.py` uses
  `"bob highlights create --include-id"` as fixture data (four occurrences).

The `research-highlights` hook spec itself is already command-agnostic: `command` is
`required` and deliberately unset. Only the docs point users at bob.

## Approach: Mermaid's official CLI (`mmdc`)

bob's check does one thing: it parses and renders each Mermaid fence with a pinned
Mermaid 12.2.0 bundle in headless Chrome. `@mermaid-js/mermaid-cli` (`mmdc`) is the
upstream tool that does exactly that. Its current release (12.0.0) depends on
`mermaid ^12.0.0`, which resolves to the same 12.2.0, so its verdicts match the PDF
pipeline bob drives for Bryan downstream. Behavior verified on athena:

- `mmdc -i check.md -o check_render.md` prints
  `Found N mermaid charts in Markdown input` (always plural), writes
  `check_render-1.svg` … `check_render-N.svg`, and exits 0.
- A parse error exits 1 with `Error: Parse error on line …` and an excerpt. It stops at
  the first failing chart and writes no SVGs.
- An installed `mmdc` can be broken for environmental reasons. athena's
  `/usr/local/bin/mmdc` 11.4.2 exits 1 with "Could not find Chrome", while
  `npx -y -p @mermaid-js/mermaid-cli mmdc` works. The macro therefore smoke-probes each
  form before trusting it, so an environment failure is never mistaken for a bad
  diagram.
- Each SVG root carries `style="max-width: <W>px"` (the natural width). That gives a
  portable replacement for bob's "figure would print below 50% size" warning: in the
  ~650 px column the macro already designs for, W above about 1300 is too wide.

Rejected alternatives: vendoring mermaid.js plus Playwright into this prompt-only plugin
as a console script (a heavy runtime dependency that duplicates mermaid-cli); remote
renderers such as mermaid.ink or kroki (they ship report content to third parties and
need the network); `@mermaid-js/parser` (it does not cover flowchart or sequence
diagrams); a configurable checker command that keeps bob as an option (macros have no
config plumbing for it, and bob would stay in the docs).

`mmdc` and `npx` stay optional host tools that the agent prompt invokes, the same way
the audio macro invokes `sase listen`. Nothing changes in `pyproject.toml`.

## Changes

Work only in the linked `sase-research-artifacts` repo. Open it with `/sase_repo`
(`sase repo open sase-research-artifacts -r "<why>"`) and read its `AGENTS.md` first.

### 1. `src/sase_research_artifacts/xprompts/research_diagrams.md`

Rewrite step 4 ("Validate") so it keeps the existing tone and density but says the
following:

1. **Pick the CLI once.** Probe `mmdc` first, then
   `npx -y -p @mermaid-js/mermaid-cli mmdc`. A form is usable only if this smoke render
   exits 0 and writes `probe.svg`:

   ```sh
   printf 'flowchart TD\n  A["ok"]\n' > <scratch>/probe.mmd
   <mmdc> -i <scratch>/probe.mmd -o <scratch>/probe.svg
   ```

   A failed probe means that form of the checker is unavailable, not that a diagram is
   bad. If a probe fails only because Chrome cannot start its sandbox (common as root or
   in containers), retry once with `-p <scratch>/puppeteer.json` holding
   `{"args": ["--no-sandbox"]}`, and keep that flag for the real check. Record
   `<mmdc> --version`.

2. **Check.** Keep the existing instruction to put every diagram in a scratch Markdown
   file with a short paragraph before each one. Then run:

   ```sh
   <mmdc> -i <scratch>/check.md -o <scratch>/check_render.md
   ```

   The check passes only when the command exits 0, its output says
   `Found N mermaid charts in Markdown input` for your N, and
   `<scratch>/check_render-1.svg` through `-N.svg` all exist. `mmdc` stops at the first
   failing diagram and prints `Parse error on line …` with an excerpt. Fix, simplify, or
   drop that diagram and rerun. Keep the existing "stop after three rounds" cap.

3. **Size.** Read each SVG's natural width with
   `grep -o 'max-width: [0-9.]*px' <svg> | head -1`. If it is above about 1300 px, the
   diagram would print below half size in the ~650 px column. Treat it like a failure:
   simplify it (fewer side-by-side boxes, or `TD` instead of `LR`) or drop it.
4. **Unavailable.** If no form is usable, propose nothing and set `status: failed` with
   the reason (step 7).

Delete every bob-specific detail: the `bob ref create` line, the `-n`/`--no-audio` note,
the "`-o` outside the library" note, the `-P` parent paragraph (Bob area or project
note, the `sase project current --json` display name, `bob capture-targets`), and "If
bob is missing".

In step 6, change the manifest example to
`check: "mmdc 12.0.0 rendered 1 of 1 Mermaid diagrams"`. Change the bullet "`check` is
bob's `diagrams:` line" to say that `check` records the CLI version and the chart count
from the passing run. Keep "rerun the check after any edit" and the byte-identical rule.

In step 7, change the `failed` example reason to
`"Mermaid CLI (mmdc) unavailable on this host"`.

Step numbers, the fail-open rules, and the manifest, label, and variable contracts stay
unchanged. The linker in `research_swarm.md` never reads `check`, so the swarm needs no
change.

> [!decision] npx_fallback = installed_only Step 4 probes only `mmdc`, with no `npx`
> form and no download. Every doc mention below drops the `npx` fallback too, and the
> new test asserts that `npx` and `@mermaid-js/mermaid-cli` are absent from the macro,
> not present.

### 2. `docs/macros.md` (`#research/diagrams` section)

- Change the manifest example `check:` line to match step 6.
- Rewrite `### Validation` to describe the new check: the probe order, the
  `mmdc -i <scratch>/check.md -o <scratch>/check_render.md` command, the pass criteria
  (exit 0, `Found N mermaid charts`, N SVGs), the ~1300 px natural-width limit, the
  three-round cap, and "no usable Mermaid CLI → propose nothing". Keep the existing
  sizing guidance (650 px column, half page, `flowchart TD`, at most four boxes side by
  side, quoted labels, no `style`/`classDef`/`click`/HTML).
- Change the `failed` JSON example reason to match step 7.

### 3. `README.md`

- In "Provider configuration", replace `command: bob highlights create --include-id`
  with a neutral placeholder such as `command: render-research-pdf`. Add one sentence:
  SASE appends the report's absolute path as the command's final argument and exports
  the project display name in `SASE_FILE_HOOK_PROJECT`, so any local renderer works.
- Change "Bob derives its PDF basename and marker id from the input Markdown basename"
  to renderer-neutral wording, for example: a renderer that names its output after the
  input basename would otherwise see digest-suffixed names.
- In the `#research/diagrams` bullet under "Macros", note the tool requirement in the
  style of the `#research/audio` bullet: it render-checks with the Mermaid CLI (`mmdc`,
  falling back to `npx`), and with neither it proposes nothing.

### 4. `docs/configuration.md`

- In "Enabling the `research-highlights` file hook", use the same placeholder and the
  same one-sentence contract note as the README.
- Change "which would leak into Bob's derived PDF basename and marker id" to the same
  renderer-neutral wording.

### 5. `docs/architecture.md`

Change "the basename-sensitive Bob Highlights command" to "a basename-sensitive local
renderer command".

### 6. `AGENTS.md` (the repo's own agent doc, not SASE memory)

In the `xprompts/` architecture bullet, add one clause: `#research/diagrams`
render-checks Mermaid with the Mermaid CLI (`mmdc`, or `npx`) and fails open without it.

### 7. Tests

- `tests/test_macro_loading.py::test_research_diagrams_restraint_check_and_anchor_rules`:
  drop the two bob assertions and the `diagrams: rendered N of N` assertion. Assert the
  new contract instead: `"mmdc -i <scratch>/check.md -o <scratch>/check_render.md"`,
  `"npx -y -p @mermaid-js/mermaid-cli mmdc"`, `"Found N mermaid charts"`, the probe
  `"<mmdc> -i <scratch>/probe.mmd -o <scratch>/probe.svg"`, and the `max-width` size
  rule. Keep the existing anchor, `grep -cF`, and byte-identical assertions.
- `tests/test_provider_specs.py`: replace all four
  `"bob highlights create --include-id"` fixture strings with `"render-research-pdf"`.
  They only prove that a local `command` survives the `use:` merge.
- Add a new guard test module, for example `tests/test_no_bob_dependency.py`. Rooted at
  `Path(__file__).resolve().parents[1]`, it reads every file under
  `src/sase_research_artifacts/` and `docs/`, plus `README.md` and `AGENTS.md`, and
  fails with each `path:line` that matches `re.compile(r"\bbob\b", re.IGNORECASE)`.
  `CHANGELOG.md` (release history) and `tests/` are out of its scope. A short docstring
  says that bob-cli is a personal tool and must never become a dependency of this
  plugin.

## Non-goals

- The linker's PDF-friendly formatting rules in `research_swarm.md` (no tables, no
  hand-numbered headings, the "Highlights PDF" wording). They are renderer-agnostic
  Markdown hygiene, never invoke bob, and stay harmless without a PDF hook.
- Renaming the `research-highlights` hook id or changing its spec description. The id is
  public config surface, and the spec stays command-agnostic.
- Illustrative `bob` mentions in other repos (the `sase` file-hook docs example,
  `sase-listen` integration prose, the `bob-cli` project names in `sase-core` test
  fixtures). None of them is a dependency.
- Editing `CHANGELOG.md`. release-please owns it. Use a `fix(research-diagrams): …`
  conventional commit message so the next release is a patch.

## Verification

1. In the `sase-research-artifacts` checkout, `sase tool run check` passes (lint plus
   tests, including the new guard test).
2. `git grep -n -i -w bob -- . ':!CHANGELOG.md' ':!tests'` prints nothing.
3. Smoke-test the literal step-4 commands in a fresh `mktemp -d` directory:
   - The probe succeeds with the first usable form.
   - A `check.md` with two valid diagrams (a `flowchart TD` and a `sequenceDiagram`)
     exits 0, prints `Found 2 mermaid charts in Markdown input`, and writes
     `check_render-1.svg` and `check_render-2.svg`.
   - The `max-width` grep prints a width for each SVG.
   - A `check.md` with a broken fence (for example `A["x" --> end`) exits 1 with
     `Parse error on line`.

   The first `npx` run may take about 30 s while it downloads mermaid-cli and Chrome.
