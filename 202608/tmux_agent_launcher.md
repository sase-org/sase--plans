---
tier: epic
status: done
title: tmux Agent — launch an interactive agent CLI in a new tmux window
goal: "Pressing `t` in Launch Control, or running `sase tmux-agent`, opens a
  keyboard-first chooser of every registered agent-CLI provider and drops the chosen one
  into a fresh, auto-numbered tmux window in the current directory. Both surfaces share
  one registry-driven catalog, so a newly registered provider plugin appears with no
  code change here.

  "
phases:
  - id: config
    title: tmux_agent configuration section
    depends_on: []
    size: small
    description:
      "config: add the `tmux_agent` config block, JSON schema entry, shipped defaults,
      and typed getters."
  - id: descriptor
    title: Interactive-CLI provider descriptor
    depends_on: []
    size: small
    description:
      "descriptor: add the `llm_interactive_cli` hook plus vendor metadata, implement it
      on every built-in provider, and expose it through the registry payload."
  - id: catalog
    title: Catalog, launch-spec, and window-name resolution
    depends_on:
      - config
      - descriptor
    size: medium
    description:
      "catalog: build the `sase.tmux_agent` package core — entry model, catalog builder,
      deterministic shortcut keys, launch-spec resolution, and window naming — as pure,
      injectable functions."
  - id: launcher
    title: tmux window launch, renumber, and menu rendering
    depends_on:
      - catalog
    size: medium
    description:
      "launcher: add the tmux command primitives that create the agent window, register
      the exit-cleanup waiter, renumber agent windows, and render the styled
      `display-menu`."
  - id: cli
    title: sase tmux-agent command
    depends_on:
      - launcher
    size: medium
    description:
      "cli: register and dispatch the `sase tmux-agent` command with its menu,
      direct-launch, list, dry-run, JSON, and internal renumber paths."
  - id: cache
    title: Catalog cache for menu latency
    depends_on:
      - cli
    size: small
    description:
      "cache: add the fingerprinted on-disk catalog cache and `-r/--refresh` so the tmux
      key binding renders in well under a quarter second."
  - id: ace
    title: Launch Control `t` and the tmux Agent panel
    depends_on:
      - launcher
    size: medium
    description:
      "ace: bind `t` in Launch Control and add the tmux Agent modal, its styles,
      footer/help wiring, and behavior tests."
  - id: polish
    title: Parity guarantee, visual snapshot, and documentation
    depends_on:
      - ace
      - cli
    size: small
    description:
      "polish: pin command parity with the shell script this feature replaces, add the
      PNG snapshot, and write the ACE, CLI, configuration, and plugin documentation."
proposed_by: bbugyi200.athena.07y
bead_id: sase-r0
create_time: 2026-09-09 19:51:52
---

- **PROMPT:**
  [prompts/202608/tmux_agent_launcher.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/tmux_agent_launcher.md)
- **BEAD:**
  [sase-r0](https://github.com/sase-org/sase--beads/blob/main/pages/sase-r0/README.md)

# Plan: tmux Agent

## Overview

The user keeps a personal shell script bound to a tmux key
(`bind A run "tmux_ai_window"`). It shows a centered, styled tmux menu of AI agent CLIs,
greys out the ones that are not installed, and — on choosing one — opens a new tmux
window named `ai` / `ai2` / `ai3` in the current pane's directory running that CLI with
the user's preferred flags. When the CLI exits, a background waiter renumbers the
remaining `ai` windows.

This epic reimplements that behavior inside SASE so that:

- the provider list comes from SASE's registered LLM provider plugins, not a hardcoded
  bash `case` statement, so a provider added later is launchable with no edit here;
- installed/not-installed detection, provider accent colors, display names, effort-flag
  translation, and install hints all reuse machinery SASE already owns;
- the same functionality is reachable from ACE (`,m` → Launch Control → `t`) and from
  the command line (`sase tmux-agent`).

### Source material

The script being ported lives in the user's `chezmoi` linked repo at
`home/bin/executable_tmux_ai_window`, with a regression test at
`tests/bash/tmux_ai_window_test.sh` and a renumber helper at
`home/bin/executable_tm-renumber-ai-windows`. Every phase agent that needs the exact
current behavior MUST read those files through the `/sase_repo` skill
(`sase repo open chezmoi -r "..."`), never by cloning or web-fetching. Nothing in this
epic modifies the chezmoi repo.

### Existing SASE machinery this epic reuses

| Need                                | Reuse                                                                                              |
| ----------------------------------- | -------------------------------------------------------------------------------------------------- |
| Registered providers                | `sase.llm_provider.registry.registered_provider_names()`, `get_llm_metadata_payload()`             |
| CLI binary name + `SASE_<P>_PATH`   | `llm_autodetect_cli_name` hook, `sase.agent_clis.detect.resolve_executable`                        |
| Installed state, version, hint      | `sase.agent_clis.operations.collect_agent_cli_statuses` → `AgentCliStatus`                         |
| Accent color                        | `sase.llm_provider.registry.provider_cli_status_color_map()`                                       |
| Display name                        | `llm_install_metadata()["display_name"]`                                                           |
| Hidden test providers               | `registry.model_picker_hidden_provider_names()` (keeps `fakey` out)                                |
| Effort → per-provider CLI flags     | `LLMProvider.invocation_option_args()` / `sase.llm_provider._effort_args.effort_cli_args`          |
| Effective default effort + override | `sase.llm_provider.config.effective_default_effort_snapshot().effective_effort()`                  |
| Provider routing disables           | `registry.build_provider_routing_statuses()`, `sase.llm_provider.get_active_provider_disables`     |
| Autodetected default provider       | `registry.get_default_provider_name()` / `get_configured_default_provider_name()`                  |
| Cache-on-disk pattern               | `sase.agent_clis.latest` (`sase_subdir(...)`, read/write/fingerprint)                              |
| tmux-window-from-a-modal pattern    | `sase.ace.tui.modals.agent_workspace_tmux_modal`, `sase.ace.tui.actions.agents._panel_tmux`        |
| Sub-modal off Launch Control        | `sase.ace.tui.modals.models_panel_provider_modal.ProviderRoutingModal` (structure, worker, styles) |

## Design decisions

These are the calls the user asked to be made deliberately. Phase agents must not
quietly reverse them.

### 1. The CLI must not use Textual, because it must run without a TTY

The script this replaces is invoked as `bind A run "tmux_ai_window"`. `tmux run-shell`
executes its command with **no controlling terminal**. Any full-screen TUI — Textual,
curses, prompt-toolkit — is impossible there. `sase tmux-agent` therefore renders its
chooser with tmux's own `display-menu`, which is a tmux _client-side_ overlay: the
process only has to emit a tmux command and exit. This keeps the command a drop-in
replacement for the existing key binding and keeps the menu identical in look and feel
to what the user has today.

### 2. ACE gets a native Textual panel, not a tmux overlay

ACE is itself a Textual application with continuously refreshing widgets. Painting a
tmux client overlay on top of it would fight ACE's repaint loop, would be invisible to
the repo's PNG snapshot suite, would look nothing like every other ACE panel, and would
break outright when ACE runs outside tmux. ACE therefore gets a real modal built the
same way `ProviderRoutingModal` and `AgentWorkspaceTmuxModal` already are.

Both surfaces call the _same_ catalog builder and the _same_ launch engine, so
"functionality" is shared and only the presentation layer differs. Where the two can
agree they must: shortcut keys, ordering, preselection, row text, and the resolved
command are computed once, in `sase.tmux_agent`, and rendered twice.

### 3. Selecting from the tmux menu calls back into `sase tmux-agent`

Menu items run `run-shell "sase tmux-agent <provider> --dir <pane-dir>"`, exactly as the
shell script re-invokes itself. Encoding the whole launch inline in the tmux command
string would need three levels of nested quoting and would have to guess the window name
before the user chooses. The callback keeps one code path for "launch this provider
here" that ACE, the menu, and a direct `sase tmux-agent claude` all share. The `cache`
phase makes that second process cheap.

### 4. Latency is a first-class requirement

Measured on this machine: importing `sase.llm_provider.registry` and building provider
routing statuses costs ~700 ms warm, while the Python interpreter plus a narrowed
argparse tree costs ~95 ms and eight `shutil.which` probes cost ~25 ms. A key binding
that takes ~0.9 s to paint a menu, then another ~0.9 s to open the window, is a visible
regression against a shell script that does it in ~50 ms. The `cache` phase stores the
expensive, slow-changing part (plugin-derived provider metadata plus merged config) on
disk behind a fingerprint, leaving PATH probes and tmux queries live. Target: menu paint
and launch callback each well under 250 ms.

### 5. Naming

`agent` alone means a SASE agent (see the `Sase Agent` / `Agent Shell` glossary terms).
What this feature launches is a raw **agent CLI**, unmanaged by SASE. User-facing text
must say "agent CLI", never a bare "agent", to avoid implying a SASE-tracked run. The
feature is named **tmux Agent** everywhere: `sase tmux-agent`, the ACE panel title
`tmux Agent`, the tmux menu title `tmux Agent`, and the docs section `tmux Agent`.

### 6. No feature flag

Read `sase/memory/sase_flags.md` before disagreeing. This epic is purely additive: `t`
is currently unbound in Launch Control, `sase tmux-agent` is a new command, and no
existing behavior changes. Each user-reaching phase (`cli`, `ace`) lands its surface
complete rather than half-built, so there is no "reaches users before it is ready"
branch to gate, and nothing here is a permanent user choice masquerading as a flag.

### 7. Provider disables annotate but do not block

Launch Control's `p` disables a provider for **SASE's automatic routing** — new
launches, follow-ups, retries, pickers. A human explicitly choosing an agent CLI to sit
in front of is not automatic routing. A disabled provider therefore stays selectable and
is annotated `routing disabled · <time> left`, so the state is visible without being a
wall.

### 8. Permission bypass is on by default, and always visible

The script bypasses approvals for every provider that supports it, and SASE's own
headless invocations do the same. `tmux_agent.bypass_permissions` defaults to `true` for
that reason, but the resolved command — including the bypass flag — is always shown in
the ACE description strip, in `--dry-run`, and in `--list -v`, and `-s/--safe` (CLI) or
`s` (ACE) launches once without it. Nothing is bypassed invisibly.

## config: tmux_agent configuration section

Add a new top-level `tmux_agent` config section. The root schema is
`additionalProperties: false`, so `src/sase/config/sase.schema.json` must be extended
before any config is readable.

### Shape

```yaml
tmux_agent:
  # Base tmux window name. The first window is this name; later ones get a
  # numeric suffix (ai, ai2, ai3, ...).
  window_name: "ai"
  # Pass each agent CLI's approval-bypass flags (see the provider's
  # llm_interactive_cli descriptor). Per-provider overrides win.
  bypass_permissions: true
  # Reasoning effort applied to launches. "" follows llm_provider.default_effort;
  # "off" passes no effort flags at all.
  effort: ""
  # Run `clear` in the new window before starting the CLI.
  clear_screen: true
  # Optional shell command run after an agent CLI window closes, alongside
  # SASE's own window renumbering. Empty means nothing extra runs.
  after_close_command: ""
  # Per-provider overrides, keyed by registered provider name.
  providers:
    claude:
      enabled: true # false hides the provider from both surfaces
      key: "" # override the single-key menu shortcut
      model: "" # pin a model, e.g. "gemini-3.7-flash-high"
      effort: "" # per-provider effort; "" inherits, "off" disables
      args: [] # extra CLI args appended verbatim
      env: {} # environment variables for the new tmux window
      bypass_permissions: true # omit to inherit the global default
```

### Work

- Add the block, with the comments above, to `src/sase/default_config.yml`. Place it
  alphabetically consistent with neighboring top-level sections.
- Add a matching `tmux_agent` object to `src/sase/config/sase.schema.json` with
  `additionalProperties: false` at both levels, `providers` as an object whose
  `additionalProperties` is the per-provider object, `effort` as an enum of
  `["", "off", "none", "minimal", "low", "medium", "high", "xhigh", "max"]`, and
  per-provider `bypass_permissions` as a plain boolean **with no default** so an absent
  key means "inherit".
- Add `src/sase/config/tmux_agent.py` following the shape of
  `src/sase/config/file_hooks.py`: frozen dataclasses `TmuxAgentConfig` and
  `TmuxAgentProviderConfig`, plus `get_tmux_agent_config()` reading through
  `load_merged_config()`. Unknown or malformed entries are dropped with a logged warning
  rather than raising — a bad config must never make the tmux key binding fail.
- Re-export the public names from `src/sase/config/__init__.py` the way the other domain
  config modules are re-exported.
- `key` must be validated as exactly one printable, non-whitespace character; anything
  else is dropped with a warning and the automatic key is used instead.

### Tests (`tests/test_config_tmux_agent.py`)

Defaults with no user config; a full user override; per-provider `bypass_permissions`
absent vs `false`; an invalid `key`; an unknown provider name in `providers`; `effort`
enum rejection surfaced by the existing schema-validation test helpers in
`tests/_config_schema_helpers.py`.

## descriptor: Interactive-CLI provider descriptor

SASE's provider plugins today describe only _headless_ invocation. Every interactive
flag in the shell script (`--dangerously-skip-permissions`, `--yolo`,
`--always-approve`, …) is currently hardcoded inside each provider's `invoke()` argv.
This phase gives providers a declarative way to describe their interactive CLI so the
launcher never hardcodes a provider name.

### New hook

Add to `LLMHookSpec` in `src/sase/llm_provider/_hookspec.py`:

```python
@hookspec(firstresult=True)
def llm_interactive_cli(self) -> dict[str, object] | None:
    """How this provider's CLI is launched interactively, in a terminal.

    All keys are optional so third-party providers stay compatible:

    ``argv``        base argv; defaults to ``[llm_autodetect_cli_name()]``.
    ``args``        always-on interactive args.
    ``bypass_args`` args that skip this CLI's approval prompts.
    ``model_args``  argv fragment selecting a model; the literal ``{model}``
                    token is replaced with the configured model exactly once.
    ``env``         environment the CLI needs in interactive mode.
    ``menu_key``    preferred single-character shortcut.
    ``supported``   ``False`` marks a provider with no interactive CLI, which
                    excludes it from the tmux Agent launcher.

    Omitting the hook entirely means "launchable as the bare CLI name with no
    extra flags", so a new provider is usable the moment it declares a CLI.
    """
```

Also document a new optional `vendor` key in `llm_install_metadata()` ("Anthropic",
"OpenAI", "Google", "Alibaba", "SST", "xAI", "Meta") — the secondary label the menu
shows. It is a descriptive field like the existing `display_name`/`docs_url`, so no new
hook is needed.

### Registry plumbing

- `provider_metadata()` in `src/sase/llm_provider/_registry_metadata.py` gains a
  normalized `"interactive_cli"` entry: unknown keys dropped,
  `argv`/`args`/`bypass_args` coerced to `tuple[str, ...]`, `env` through
  `normalize_str_dict`, `menu_key` truncated to one character, `supported` defaulting to
  `True`. Malformed values degrade to the default rather than raising — the existing
  `_call_optional` pattern already swallows plugin exceptions and this must match.
- Pull `vendor` out of the install metadata in `_install_metadata()`.
- Add `registry.provider_interactive_cli_map()` and `registry.provider_vendor_map()`
  next to `provider_cli_status_color_map()`, both reading the memoized payload.

### Built-in provider descriptors

Implement `llm_interactive_cli` on the seven real providers so behavior matches the
shell script exactly. `menu_key` values are the script's, preserving muscle memory.

| Provider   | `menu_key` | `bypass_args`                                    | `vendor`    |
| ---------- | ---------- | ------------------------------------------------ | ----------- |
| `claude`   | `c`        | `["--dangerously-skip-permissions"]`             | Anthropic   |
| `codex`    | `x`        | `["--dangerously-bypass-approvals-and-sandbox"]` | OpenAI      |
| `agy`      | `a`        | `["--dangerously-skip-permissions"]`             | Antigravity |
| `qwen`     | `q`        | `["--yolo"]`                                     | Alibaba     |
| `opencode` | `o`        | `[]`                                             | SST         |
| `grok`     | `g`        | `["--always-approve"]`                           | xAI         |
| `muse`     | `m`        | `["--yolo"]`                                     | Meta        |

`opencode` gets **no** bypass args: the script launches it bare, and its headless argv
is not evidence that the interactive CLI accepts the same flag. A phase agent must not
add a bypass flag it cannot show is accepted interactively; users can add one through
`tmux_agent.providers.opencode.args`.

All seven declare `model_args: ["--model", "{model}"]` (verified: every built-in
provider's headless argv uses `--model <value>`). `muse` additionally declares
`env: {"MUSE_NO_AUTO_UPDATE": "1"}`, mirroring the reason already documented in
`muse.py`: a multi-hour session must not have its binary swapped mid-flight.

`fakey` declares `supported: False`.

`EDITOR=nvim` from the script is a personal preference, not a provider requirement, and
belongs in `tmux_agent.providers.claude.env` — not in the plugin.

### Tests (`tests/llm_provider/test_interactive_cli_metadata.py`)

Every registered provider's descriptor normalizes; a provider that omits the hook falls
back to its CLI name; malformed hook payloads (wrong types, non-string members, a
multi-character `menu_key`, an exception) degrade without raising; `menu_key` values are
unique across built-ins — mirroring the uniqueness assertion the shell script's own
bashunit test pins.

## catalog: Catalog, launch-spec, and window-name resolution

Create `src/sase/tmux_agent/` — the single engine both front ends call. It lives in
Python, not in the sibling Rust core, because its inputs are pluggy provider plugins and
`LLMProvider.invocation_option_args()`, which the Rust core cannot reach;
`src/sase/agent_clis/` is the established precedent for registry-driven logic shared by
a CLI and the TUI.

Everything in this phase is pure and injectable: no module reaches for `subprocess`,
`os.environ`, or the clock without a caller-supplied seam, so both front ends and the
tests drive identical code.

### `models.py`

```python
@dataclass(frozen=True)
class TmuxAgentEntry:
    provider: str            # registry key, e.g. "claude"
    display_name: str        # "Claude Code"
    vendor: str              # "Anthropic" (may be "")
    color: str               # "#D97757"
    key: str                 # resolved single-character shortcut
    binary: str              # "claude"
    executable: str | None   # resolved path, None when not installed
    installed: bool
    install_hint: str        # from AgentCliStatus, shown when not installed
    routing_disabled: TemporaryProviderDisable | None
    argv: tuple[str, ...]    # fully resolved launch argv
    env: tuple[tuple[str, str], ...]
    effort: str | None       # effort actually applied, None when none was
    effort_skipped: str | None  # level requested but unsupported, else None
    bypass: bool             # whether bypass args are in argv


@dataclass(frozen=True)
class TmuxAgentCatalog:
    entries: tuple[TmuxAgentEntry, ...]   # menu order
    default_provider: str | None          # preselected entry
    directory: str                        # launch directory
```

### `launch_spec.py`

`resolve_launch_argv(provider, *, descriptor, provider_config, catalog_config, effort, provider_obj)`
composes, in order:

1. `descriptor.argv` (or `[cli_name]`),
2. `descriptor.args`,
3. `descriptor.bypass_args` when bypass is on,
4. `descriptor.model_args` with `{model}` substituted, when a model is pinned,
5. effort args from `provider_obj.invocation_option_args(LLMInvocationOptions(...))`,
6. `provider_config.args`.

Effort resolution: per-provider `effort` wins over `tmux_agent.effort`, which wins over
`llm_provider.default_effort` via
`effective_default_effort_snapshot().effective_effort()` so an active temporary effort
override is honored. `"off"` at either level means no effort args. An effort coming from
config is **not** explicit, so `effort_cli_args` logs-and-skips an unsupported level;
the skipped level is recorded on the entry as `effort_skipped` and surfaced in the UI
rather than swallowed. An effort passed explicitly on the command line **is** explicit,
so an unsupported level raises and the CLI reports it as a usage error.

Practical consequence to document: `tmux_agent.effort: "max"` gives Claude
`--effort max` and Muse `--reasoning-effort ultra`, but Codex tops out at `xhigh`, so a
user who wants the script's exact Codex behavior sets
`tmux_agent.providers.codex.effort: "xhigh"`.

A pinned model on a provider whose descriptor declares no `model_args` is a config
error: drop the pin, warn once naming the provider and the config path.

`env` merges `descriptor.env` under `provider_config.env` (user wins).

### `keys.py`

`assign_menu_keys(providers)` is deterministic and stable:

1. `tmux_agent.providers.<name>.key` when free;
2. the descriptor's `menu_key` when free;
3. the first free lowercase letter of the provider name, then of the display name;
4. the first free digit `1`–`9`, then any remaining free lowercase letter;
5. otherwise no key — the row is still selectable by navigating to it.

Reserved and never auto-assigned: `j` and `k`, which navigate in both surfaces. `q` is
_not_ reserved, so `qwen` keeps `q` exactly as it does today; the panel and menu titles
both state that `Esc` closes, and `q` closes only when unclaimed. Providers are
processed in registry (alphabetical) order so assignment never depends on iteration
accidents. Final menu order is by assigned key, then provider name — the same ordering
rule the shell script uses, so appending a provider never shuffles the existing rows.

### `catalog.py`

`build_tmux_agent_catalog(*, directory, statuses=None, now=None)` assembles entries from
`collect_agent_cli_statuses(offline=True)` (never touch the network on this path),
`provider_cli_status_color_map()`, `provider_vendor_map()`,
`provider_interactive_cli_map()`, `get_active_provider_disables(now)`, and
`get_tmux_agent_config()`.

Excluded: providers in `model_picker_hidden_provider_names()`, providers whose
descriptor sets `supported: False`, providers whose config sets `enabled: false`, and
providers that declare no CLI name at all.

`default_provider` is the first of: the configured `llm_provider.provider` when
installed; the highest-`llm_autodetect_priority` installed provider; the first installed
entry; `None` when nothing is installed.

### `window.py` (naming only; the tmux calls land in `launcher`)

`next_window_name(base, existing)` reproduces the script's rule exactly: if no window
matches `^<base>[0-9]*$`, return `base`; otherwise return `<base><n>` for the smallest
`n >= 2` not taken. `renumber_plan(base, windows)` takes `(index, name)` pairs sorted by
window index and returns the `(index, new_name)` renames needed so matching windows read
`base`, `base2`, `base3`, … — the port of `tm-renumber-ai-windows`, as a pure function.

### Tests (`tests/tmux_agent/`)

`test_keys.py` (stability, collisions, the reserved set, a provider with no free
letters), `test_launch_spec.py` (each composition step, effort precedence and skip
recording, explicit-effort raising, model pin without `model_args`, env merge order),
`test_catalog.py` (exclusions, ordering, default preselection, disabled-routing
annotation, nothing-installed), `test_window.py` (naming and renumber plans, including
gaps and already-correct sequences).

## launcher: tmux window launch, renumber, and menu rendering

Add the side-effecting half of the package. Every tmux invocation goes through one
injectable runner so tests never need a live tmux server.

### `tmux.py`

A thin, typed wrapper over `tmux(1)`: `TmuxRunner` with a
`run(args) -> CompletedProcess` seam, plus `tmux_available()`, `inside_tmux()` (`$TMUX`
or `$TMUX_PANE`, matching `sase.ace.tui.graphics._viewer_tmux_common.is_tmux_session`),
`tmux_version()`, `current_pane_directory()`
(`display-message -p '#{pane_current_path}'`), `list_windows()`
(`list-windows -F '#{window_index}:#{window_name}'`), `new_window(...)`,
`rename_window(...)`, `run_shell_background(...)`, and `display_menu(...)`.

`tmux_version()` reuses the parsing already written for the doctor check in
`src/sase/doctor/checks_deep_terminal.py`. Promote that parser into `tmux.py` and have
the doctor check import it rather than duplicating the regex — do not leave two copies,
and do not import the doctor module's private helper from here.

### `launch.py`

`launch_agent_window(entry, *, directory, config, runner)`:

1. Resolve the window name from `list_windows()` and `next_window_name()`.
2. Mint a cleanup channel `sase-tmux-agent-<uuid4().hex[:12]>`.
3. `tmux run-shell -b "tmux wait-for <channel> && sase tmux-agent --renumber"`,
   appending ` && <after_close_command>` when configured. Registering the waiter
   _before_ creating the window is deliberate and matches the script; the `-b` form
   hands the job to the tmux server so it survives the window closing.
4. `tmux new-window -n <name> -c <directory> [-e K=V ...] '<shell-command>'` where the
   shell command is `clear; <shlex-joined argv>; tmux wait-for -S <channel>` (the
   `clear;` prefix only when `clear_screen` is true). Build it with `shlex.quote` per
   argv element — never with string interpolation.
5. Return a `TmuxAgentLaunch` result carrying the window name, channel, argv, and the
   directory, so callers can report exactly what happened.

Errors return a typed failure rather than raising: tmux missing, not inside tmux, the
directory does not exist, the provider is not installed, `new-window` returned non-zero.
Each carries a one-sentence, actionable message.

### `renumber.py`

`renumber_agent_windows(*, config, runner)` reads `list_windows()`, applies
`renumber_plan()`, and issues the renames. Idempotent, and a no-op when nothing matches.

### `menu.py`

`build_display_menu_args(catalog, *, title, self_command)` returns the exact
`tmux display-menu` argv:

- `-b rounded`, `-x C`, `-y C`, `-C <default index>`;
- `-T "#[align=centre,fg=…,bg=…,bold] tmux Agent "`, plus styled `-S` / `-s` / `-H`;
- one triple per entry. Installed: name formatted
  `#[fg=<accent>,bold]<display><pad>#[fg=#565f89,nobold] <vendor>`, key = the assigned
  key, command = `run-shell '<self_command> <provider> --dir <dir>'`. Not installed: the
  name is prefixed with `-` so tmux renders it dim and unselectable, and both key and
  command are empty — the script's convention, preserved.
- `self_command` is derived from `sys.executable` + `-m sase` when `sase` is not
  resolvable on `PATH`, so the menu keeps working from an editable checkout, mirroring
  how `src/sase/main/ace_tmux.py` builds its relaunch command.

Style flags `-b`, `-s`, `-S`, and `-H` are gated on tmux ≥ 3.4; on older tmux the same
menu is emitted without them. The menu never silently fails: if `display-menu` returns
non-zero, the caller reports the stderr.

Colors come from the catalog entries (provider accent) and from one small palette
constant module shared with the ACE panel, so the two surfaces cannot drift.

### Tests

`tests/tmux_agent/test_launch.py` and `test_menu.py` with a fake runner asserting the
exact argv of every tmux call: waiter registered before the window, correct
`-c`/`-n`/`-e` flags, quoting of a directory with a space and a single quote, the
`clear;`/`wait-for -S` shape, `--renumber` callback, `after_close_command` appended.
`test_renumber.py` asserts the rename calls for gapped, reversed, and already-correct
window sets. `test_menu.py` asserts the disabled-row `-` prefix, key ordering, the
default-choice index, and the tmux < 3.4 degraded form.

## cli: sase tmux-agent command

### Surface

```
sase tmux-agent [<provider>] [-c <dir>] [-e <level>] [-j] [-l] [-n] [-r] [-s] [-v]
```

| Form                     | Behavior                                                                    |
| ------------------------ | --------------------------------------------------------------------------- |
| `sase tmux-agent`        | Inside tmux: paint the `display-menu`. Outside: the actionable error below. |
| `sase tmux-agent claude` | Launch that provider directly in a new window.                              |
| `-c`, `--dir <dir>`      | Launch directory; defaults to the current pane's path, else `$PWD`.         |
| `-e`, `--effort <level>` | Explicit effort for this launch; unsupported levels are a usage error.      |
| `-j`, `--json`           | Versioned JSON envelope of the catalog or the dry-run plan.                 |
| `-l`, `--list`           | Print the catalog as a Rich table; works outside tmux.                      |
| `-n`, `--dry-run`        | Print the window name, directory, env, and exact command; change nothing.   |
| `-r`, `--refresh`        | Rebuild the catalog cache before doing anything else.                       |
| `-s`, `--safe`           | Launch without the provider's approval-bypass args.                         |
| `-v`, `--verbose`        | With `--list`, add resolved paths, full commands, and install hints.        |
| `--renumber`             | Internal: renumber agent windows. Invoked by the exit waiter.               |

No subcommands. In particular there is deliberately **no `list` subcommand**:
`_default_list_subcommands()` in `src/sase/main/parser.py` makes a bare group invocation
delegate to a `list` child and print a delegation notice, which would break the bare
`sase tmux-agent` menu that the tmux key binding depends on. `--list` is an option for
that reason, and the parser description must say so.

`--renumber` is an internal subprocess argument and is exempt from the short-alias rule
in `sase/memory/cli_rules.md`; every other long option above has one. The provider is a
positional because it is the one value the command can require of the user.

Read `sase/memory/cli_rules.md` with `/sase_memory_read` before writing the parser, and
make `-h` output excellent: sorted options, a real description, and an `examples:`
epilog covering the bare menu, a direct launch, `--list`, `--dry-run`, and the tmux
binding (`bind A run "sase tmux-agent"`).

### Files

- `src/sase/main/parser_tmux_agent.py` — `register_tmux_agent_parser`.
- `src/sase/main/parser.py` — one `_COMMAND_REGISTRARS` entry, alphabetically placed.
- `src/sase/main/parser_full_registrars.py` — import and register.
- `src/sase/main/entry.py` — dispatch block, alphabetically placed with the others.
- `src/sase/main/tmux_agent_handler.py` — thin dispatcher, in the shape of
  `agent_cli_handler.py`, deferring imports.
- `src/sase/tmux_agent/cli.py` — the Rich rendering and exit codes, in the shape of
  `src/sase/agent_clis/cli_list.py`: a `TMUX_AGENT_JSON_SCHEMA_VERSION`, a table with
  provider-accent-colored names, an installed/not-installed cell, the shortcut key, and
  a summary line.

### Behavior details

- Outside tmux with no `--list`/`--dry-run`/`--json`: exit 2 with
  `sase tmux-agent: not inside a tmux session; start tmux first, or use --list to see available agent CLIs.`
  — and then print the list anyway, because a dead end is worse than a hint.
- tmux missing entirely: exit 2 naming the missing binary.
- Unknown provider: exit 2 listing the known providers and, when a close match exists,
  suggesting it (`AgentCliUnknownName` already models this shape).
- Not installed: exit 1 with the provider's `install_hint`.
- Success: print `sase_tmux_agent_window=<name>` and `sase_tmux_agent_provider=<name>`
  on stdout, following the `key=value` convention `src/sase/main/ace_tmux.py` already
  established for scriptable tmux launches.
- `--renumber` never prints on success and never fails loudly; it is a background hook.

### Tests (`tests/tmux_agent/test_cli.py`, `tests/main/`)

Argv parsing for every form; the menu path with a fake runner; direct launch;
outside-tmux error text and exit code; unknown and not-installed providers; `--dry-run`
output shape; `--json` envelope keys and schema version; `--renumber`; and a parser test
asserting `sase tmux-agent` has no `list` child so the delegation default can never
attach to it.

## cache: Catalog cache for menu latency

Add `src/sase/tmux_agent/cache.py`, modeled on `src/sase/agent_clis/latest.py`.

- Path: `sase_subdir("tmux_agent") / "catalog_cache.json"`, written atomically.
- Payload: the provider-derived, slow-to-compute half of the catalog — display name,
  vendor, color, binary name, descriptor, assigned key — plus the resolved `tmux_agent`
  config and the effective effort. Never cached: `shutil.which` results, provider
  disables, the pane directory, or the window list. Those are cheap and change often, so
  they stay live and a stale cache can never claim an uninstalled CLI is installed.
- Fingerprint: SASE version, the sorted `(name, value)` pairs of the `sase_llm` entry
  points (~30 ms), the `(path, mtime_ns, size)` of every config layer that contributed,
  and the schema version of the cache file itself. Any mismatch rebuilds and rewrites.
- `-r/--refresh` forces a rebuild. A corrupt or unreadable cache is treated as a miss,
  never as an error.

Wire it into the CLI menu and launch paths only. ACE already loads its catalog on a
Textual worker thread and repaints when it lands, so it reads through the same helper
but does not depend on a hit.

### Tests (`tests/tmux_agent/test_cache.py`)

Hit, miss, and each fingerprint component changing independently; corrupt JSON; a
read-only cache directory; and an assertion that installed-state is absent from the
cache payload.

## ace: Launch Control `t` and the tmux Agent panel

### Binding

Add `("t", "tmux_agent", "tmux Agent")` to `ModelsPanel.BINDINGS` in
`src/sase/ace/tui/modals/models_panel.py`. This is a fixed panel binding like `p` and
`ctrl+r`, not a leader keymap, so `src/sase/default_config.yml` needs no change. `t` is
free: it is unused by `ModelsPanel`, by `OptionListNavigationMixin.NAVIGATION_BINDINGS`,
and by jump mode, which intercepts keys ahead of modal bindings while active.

Add `[green]t[/green]=tmux Agent` to every branch of `_footer_markup()` in
`models_panel_display.py` that already advertises `p=Providers`, and add the row to the
Launch Control key table in the help modal wiring if one is present for that panel.

### Mixin

`src/sase/ace/tui/modals/models_panel_tmux_agent.py` — a `ModelsPanelTmuxAgentMixin`
built like `ModelsPanelProvidersMixin`:

- `action_tmux_agent()` warns and returns when ACE is not running inside tmux
  (`"ACE is not running inside tmux; start ACE in a tmux window to launch agent CLIs."`).
- Otherwise it pushes `TmuxAgentModal`, seeded with an empty catalog and a
  `load_catalog` callable so the expensive build happens on a Textual worker thread and
  never blocks the UI — the pattern `ModelsPanelProvidersMixin` already uses for the
  provider snapshot. Read `sase/memory/tui_perf.md` with `/sase_memory_read` before
  touching this: no blocking I/O on the message loop.
- Mix it into `ModelsPanel` and cancel its worker from `on_unmount`.

### Modal

`src/sase/ace/tui/modals/tmux_agent_modal.py` —
`TmuxAgentModal(OptionListNavigationMixin, ModalScreen[bool])`, structured exactly like
`ProviderRoutingModal`:

- `compose()`: container, title, summary, `OptionList`, description strip, footer.
- Title: `tmux Agent` plus `· <N> agent CLIs · <compacted directory>`, using the
  `~`-relative helper style from `agent_workspace_tmux_modal._compact_path`.
- Summary: `Opens a new tmux window in this directory.`
- Row grid, echoing the tmux menu so the two surfaces read the same:
  `<key>  ● <display name>  <vendor>  <state>` — the key in the selector accent, the
  bullet and display name in the provider's accent color, the vendor dim, and the state
  one of `ready`, `not installed`, or `routing disabled · <time> left`. Not-installed
  rows render dim and are `disabled=True` in the `OptionList`, matching the tmux menu's
  unselectable rows and reusing `ProviderRoutingModal`'s `_first_enabled_option_index` /
  `_option_is_disabled` helpers.
- Two-line description strip for the highlighted row: line one is the exact command that
  will run, line two is the window name that will be created plus effort and bypass
  state — or, for a not-installed row, its `install_hint`, and for a skipped effort, the
  level that was dropped and why.
- Keys: `enter` / the row's assigned key → launch; `s` → launch without bypass args;
  `j`/`k`/arrows/`ctrl+n`/`ctrl+p` → navigate; `esc` → close, and `q` → close only when
  no provider claimed `q`. Footer states this.
- Launching runs `launch_agent_window` on a worker, then notifies
  `Opened tmux window: <name> · <display name>` and dismisses back to Launch Control —
  the same return-to-parent behavior `ProviderRoutingModal` has. A failure notifies with
  `severity="error"` and leaves the modal open.
- Register the modal in `src/sase/ace/tui/modals/__init__.py` and `__init__.pyi` the way
  `AgentWorkspaceTmuxModal` is registered.

### Styles

Add a `#tmux-agent-*` block to `src/sase/ace/tui/styles.tcss`, copying the geometry of
the `#provider-routing-*` block at line ~2415 so the panel sits identically on screen.

### Tests (`tests/ace/tui/test_tmux_agent_modal.py`, `tests/test_models_panel_keymaps.py`)

`t` opens the modal and is listed in the footer; `t` outside tmux warns instead;
navigation skips not-installed rows; a selector key launches; `enter` launches; `s`
launches without bypass; the description strip shows the exact command; a launch failure
keeps the modal open with an error toast; the catalog worker is cancelled on unmount.

## polish: Parity guarantee, visual snapshot, and documentation

### Parity test

`tests/tmux_agent/test_shell_script_parity.py` pins the port against the script it
replaces. With the documented parity config applied and all seven CLIs marked installed,
the resolved argv for each provider must equal the script's:

```yaml
tmux_agent:
  effort: "max"
  providers:
    claude: { env: { EDITOR: nvim } }
    codex: { effort: "xhigh" }
    grok: { effort: "xhigh" }
    opencode: { effort: "off" }
    agy: { model: "gemini-3.7-flash-high" }
    qwen: { model: "qwen3.6-plus" }
    muse: { model: "muse-spark-1.2" }
```

| Provider   | Expected argv                                                                        |
| ---------- | ------------------------------------------------------------------------------------ |
| `claude`   | `claude --dangerously-skip-permissions --effort max`                                 |
| `codex`    | `codex --dangerously-bypass-approvals-and-sandbox -c model_reasoning_effort="xhigh"` |
| `agy`      | `agy --dangerously-skip-permissions --model gemini-3.7-flash-high`                   |
| `qwen`     | `qwen --yolo --model qwen3.6-plus`                                                   |
| `opencode` | `opencode`                                                                           |
| `grok`     | `grok --always-approve --effort xhigh`                                               |
| `muse`     | `muse --yolo --model muse-spark-1.2 --reasoning-effort ultra`                        |

The table is written in `launch_spec.py`'s composition order, which differs from the
script's hand-written order for `agy`, `qwen`, `grok`, and `muse`. Flag _order_ is not
part of the contract; the test compares the leading binary plus the flag _set_.

Three per-provider entries in that config are load-bearing and must be explained in
`docs/configuration.md` rather than left as magic:

- `codex` and `grok` cap out at `xhigh`. A config-default effort is best-effort, so
  `effort_cli_args` logs and skips `max` rather than downgrading it — without the
  per-provider `xhigh` they would launch with no effort flag at all, unlike the script.
- `opencode` accepts every level as `--variant <level>`, so a global `max` would add a
  flag the script never passes; `effort: "off"` keeps it bare.
- `agy` and `qwen` need no entry: both declare an empty supported-effort map, so the
  global `max` is skipped for them automatically and their argv already matches.

Also assert the menu keys are `c/x/a/q/o/g/m`, the window naming sequence is `ai`,
`ai2`, `ai3`, and the renumber plan matches `tm-renumber-ai-windows` for the same
inputs. Read the script and its bashunit test through `/sase_repo` when writing this.

### Visual snapshot

Add a PNG snapshot of the tmux Agent panel to `tests/ace/tui/visual/`, following
`test_ace_png_snapshots_models_panel_modals.py`. Cover one frame with a mix of ready,
not-installed, and routing-disabled rows so the accent colors, the dim disabled row, and
the description strip are all pinned. Goldens go in
`tests/ace/tui/visual/snapshots/png/`; generate with `--sase-update-visual-snapshots`
and verify with `just test-visual`.

### Documentation

- `docs/ace.md`: add `t` to the three Launch Control key tables and write a
  `### tmux Agent` subsection under Launch Control covering the row grid, keys, the
  `q`-is-claimable rule, the not-installed and routing-disabled states, and the
  not-inside-tmux warning.
- `docs/cli.md`: one command-table row for `sase tmux-agent`, linking to the new
  section.
- `docs/configuration.md`: the full `tmux_agent` block with every field, defaults, and
  the parity recipe above.
- `docs/plugins.md`: document `llm_interactive_cli` alongside the other LLM metadata
  hooks, and the new optional `vendor` key on `llm_install_metadata`.
- `docs/agent_providers.md`: a short pointer that installed provider CLIs are launchable
  interactively with `sase tmux-agent`.

Do **not** touch `sase/memory/*.md`, `AGENTS.md`, or the generated provider instruction
shims; no phase in this epic has permission to.

## Verification

Every phase runs `just install` first (workspaces are ephemeral) and `just check` before
finishing. The `polish` phase, and the epic's combined tree, run `just check-full`
through `/sase_monitor` with a `--next` action — never inline. The `ace` and `polish`
phases also run `just test-visual`.

Symvision will flag the new package's private helpers if they are imported across module
boundaries; read `sase/memory/symvision.md` with `/sase_memory_read` before adding a
pragma or an epic whitelist entry.

## Out of scope

- Resolving the launched CLI's model through SASE's `@alias` system. The launcher
  chooses a _provider_; a model is an optional per-provider pin. Alias-driven selection
  is a plausible follow-up, not part of this epic.
- Registering the launched CLI as a SASE agent. These windows are unmanaged, exactly as
  they are today; nothing appears in `sase agent list`.
- Splitting panes, reusing an existing agent window, or session-level layout management.
- Changing, deleting, or deploying anything in the chezmoi repo, including retiring
  `tmux_ai_window` or rebinding `prefix + A`. That is the user's call once this lands.
