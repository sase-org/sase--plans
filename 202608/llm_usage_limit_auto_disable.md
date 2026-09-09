---
tier: epic
title: Auto-disable LLM providers on usage-limit errors
goal: "When a sase agent fails because its LLM provider reported a usage/quota limit,
  sase recognizes that failure, temporarily disables just that provider (honoring the
  reset time the provider itself reported when one is available), stops wasting retries
  on it, and tells the user what happened with a rich notification. Defaults work out of
  the box for every shipped provider, and users can extend or replace the patterns and
  durations from config.

  "
phases:
  - id: detect
    title: Usage-limit detection core
    depends_on: []
    size: medium
    description: "detect: add the `llm_provider.usage_limit` config section, its JSON
      schema, the `llm_default_usage_limit_config` plugin hook, evidence-based built-in
      patterns for every shipped provider, and the normalize/match/exclude plus
      reset-hint parsing logic, with a regression corpus of real captured provider
      messages.

      "
  - id: enforce
    title: Runtime disable and retry precedence
    depends_on:
      - detect
    size: medium
    description: "enforce: call detection from the LLM invocation error paths, write the
      temporary provider disable through the existing Rust-backed store with a
      structured source, make a usage-limit failure take precedence over the retry loop
      so agents stop sleeping through futile waits, and record telemetry.

      "
  - id: notify
    title: Rich usage-limit notification
    depends_on:
      - enforce
    size: small
    description: "notify: add a notification sender that reports which provider was
      disabled, for how long and until when, what the provider actually said, and which
      agent tripped it, with once-per-disable-window dedup.

      "
  - id: surface
    title: Surface the disable reason and document the feature
    depends_on:
      - enforce
    size: small
    description:
      "surface: render automatic versus manual provenance in the Models panel and the
      top-bar provider-disable indicator, and document the new config section in the
      default config and user docs."
proposed_by: bbugyi200.athena.03j
bead_id: sase-n4
create_time: 2026-09-09 19:50:48
status: wip
---

- **PROMPT:**
  [prompts/202608/llm_usage_limit_auto_disable.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/llm_usage_limit_auto_disable.md)
- **BEAD:**
  [sase-n4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-n4/README.md)

# Plan: Auto-disable LLM providers on usage-limit errors

## Why

Today, when a provider reports a usage limit, every sase agent routed to that provider
keeps failing. Worse, the retry subsystem actively makes it worse: the `codex` built-in
retry patterns include `"rate limit"` and `"429 Too Many Requests"`, so a usage-limit
failure is classified as _retryable_ and the agent sleeps `60s`, then `300s`, then
`1800s` before failing anyway. Meanwhile every other agent launched during the outage
repeats the same cycle.

sase already has the exact mechanism needed to fix this, and it is fully wired:
`TemporaryProviderDisable` (`src/sase/llm_provider/provider_disable.py`, backed by
`sase_core::provider_disable`) is a machine-wide, self-expiring provider disable that
routing, model aliases, `%model` completion, and the ACE Models panel already respect.
It has exactly one writer today: a human clicking through the ACE Models panel with
`source="ace"`.

This epic makes provider failures write that same record automatically.

## Research: what providers actually say

The built-in patterns in this plan are not guesses. They come from two sources: real
historical sase failures, and the shipped provider CLI binaries.

### Confirmed from real sase agent failures

Found in `~/.sase/chats/2026*/...-workflow_*_ERROR-*.md`, which record the failing
agent's model plus the captured `stderr`/`output`:

| Provider | Date                                   | Captured message                                                                                                                                                           |
| -------- | -------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `claude` | 2026-08-15 (6 separate agent failures) | `You've hit your weekly limit · resets 8pm (America/New_York)`                                                                                                             |
| `grok`   | 2026-08-14 (2 separate agent failures) | `Error: You’ve reached your free Grok Build usage limit for now. Get SuperGrok for much higher limits, or try again later: https://grok.com/supergrok?referrer=grok-build` |

Both surfaced through `_invoke.py` as:

```
Error running LLM provider command (exit code 1)
stderr: [result] You've hit your weekly limit · resets 8pm (America/New_York)
output: You've hit your weekly limit · resets 8pm (America/New_York)
```

### Confirmed from the shipped provider CLIs

Extracted from the installed binaries/bundles:

- **claude** (`~/.local/share/claude/versions/2.1.233`) builds the message from a
  template `You've hit your ${label}` where `label` comes from a fixed map:
  `five_hour: "session limit"`, `seven_day: "weekly limit"`,
  `seven_day_opus: "Opus limit"`, `seven_day_sonnet: "Sonnet limit"`, plus overage
  variants. An optional ` · resets ${when}` suffix is appended.
- **codex** (`@openai/codex` native binary) ships these literals:
  `You've hit your usage limit.`, `You've hit your usage limit for `,
  `You've hit your usage limit. Upgrade to Plus to continue using Codex (…)`,
  `You've hit your usage limit. Upgrade to Pro (…)`,
  `You've hit your usage limit. Visit …/settings/usage to purchase more credits`,
  `You've hit your usage limit. To get more access now, send a request to your admin`.
- **grok** (`~/.grok/downloads/grok-1.0.4-linux-x86_64`) matches the captured live
  message above.
- **qwen** (`@qwen-code/qwen-code`) has no distinctive prose limit message; its limit
  failures surface as transport-level `429` / `Rate limit exceeded` /
  `RESOURCE_EXHAUSTED`.
- **agy** is a Go binary built on Gemini Code Assist (`code_assist_client`,
  `FetchQuotaStatus`), so its limit failures surface as Google API quota errors:
  `RESOURCE_EXHAUSTED`, `Quota exceeded`.

### Three hazards this research exposed

These are the reason the detection layer is not a one-line substring check.

1. **Apostrophes differ across providers.** `claude` and `codex` emit an ASCII
   apostrophe (`0x27`) in `You've`; `grok` emits U+2019 (`’`, bytes `e2 80 99`) in
   `You’ve`. Both were verified by hexdump. A literal pattern list will silently
   half-work unless the matcher normalizes first.
2. **Claude ships near-miss strings that must NOT disable anything.** The same binary
   contains `[Usage limit approaching. Checkpoint now: …]`,
   `[Usage limit reached — grace window active. Wrap up: …]`,
   `Approaching ${limit} · resets ${when}`, and
   `Fast limit reached and temporarily disabled · resets in ${t}`. The first three are
   advisory text injected into a _successful_ run; the fourth is a fast-mode cooldown,
   not an account limit. Matching `"usage limit"` naively against agent output would
   disable Claude for 24h because an agent was _warned_ it was approaching a limit.
3. **Usage-limit patterns overlap existing retry patterns.** `codex`'s retry config
   already claims `"rate limit"` and `"429 Too Many Requests"`. Ordering between the two
   subsystems must be explicit, not incidental.

## Design decisions

**Detection stays in Python; state stays in Rust.** Per the Rust core boundary rule,
shared backend behavior belongs in `../sase-core`. The disable _state_ already does, and
this epic reuses it unchanged — no `sase-core` change is required. The _detection_ half
reads a Python subprocess failure inside the Python agent runner and is the direct
sibling of `retry_config.py`; no other frontend re-derives it, because every frontend
consumes the resulting disable record that Rust already owns. Implementing agents should
not port pattern matching into `sase-core`.

**Mirror the retry subsystem's config ergonomics.** `ProviderRetryConfig` is the model
to follow: plugin-supplied built-in defaults via a pluggy hook, user config merged on
top, list fields additive with dedup, scalars overriding by key presence. Users get good
behavior with zero config and can extend it without retyping the built-ins.

**Prefer the provider's own reset time over a fixed duration.** Claude and Codex both
tell us when the limit resets. Disabling until the real reset is strictly better than a
flat 24h guess, and it is what makes this feature feel considered rather than blunt. The
fixed default is the fallback, not the primary path.

**Never let this subsystem break an agent run.** Detection runs inside an exception
handler on the failure path. Every entry point must be wrapped so that a bug in pattern
matching, reset parsing, disable writing, or notification can never replace or mask the
provider error the agent actually needs to see.

---

## Usage-limit detection core

Phase `detect`. No dependencies.

### New module

Add `src/sase/llm_provider/usage_limit_config.py`, deliberately parallel to
`retry_config.py`.

```python
@dataclass
class ProviderUsageLimitConfig:
    patterns: list[str] = field(default_factory=list)
    exclude_patterns: list[str] = field(default_factory=list)
    disable_seconds: int | None = None      # None => fall back to global default
    honor_reset_hint: bool | None = None    # None => fall back to global default
```

Plus a resolved global view:

```python
@dataclass
class UsageLimitSettings:
    enabled: bool = True
    disable_seconds: int = 86400            # 24h, per the request
    min_disable_seconds: int = 60
    max_disable_seconds: int = 604800       # 7d cap on any parsed reset
    honor_reset_hint: bool = True
    notify: bool = True
```

And the detection result, which is what the `enforce` and `notify` phases consume:

```python
@dataclass(frozen=True)
class UsageLimitDetection:
    provider: str
    matched_pattern: str
    message: str            # normalized, truncated trigger snippet
    raw_message: str        # untruncated original, for the notification body
    disable_seconds: float
    expires_at: float | None
    reset_hint: str | None  # e.g. "8pm (America/New_York)" when parsed
    used_reset_hint: bool
```

### Config shape

Under `llm_provider`, alongside `retry`:

```yaml
llm_provider:
  usage_limit:
    enabled: true
    disable_seconds: 86400 # default 24h when no reset time is available
    min_disable_seconds: 60
    max_disable_seconds: 604800 # clamp for any parsed reset time
    honor_reset_hint: true
    notify: true
    providers:
      claude:
        patterns: [] # ADDED to the built-ins by default
        exclude_patterns: [] # ADDED to the built-ins by default
        disable_seconds: null # optional per-provider override
        honor_reset_hint: null
        replace_patterns: false # true => user patterns REPLACE the built-ins
```

This diverges from `retry`'s flat `retry.<provider>` map only by nesting per-provider
entries under `providers:`, because unlike `retry` this feature needs genuine global
scalars. Document that rationale in `default_config.yml`.

Merge rules, matching `retry_config._merge_with_built_in`:

- `patterns` and `exclude_patterns` are **additive**: built-in first, then user, deduped
  preserving first-seen order. This is the "add to the configuration" path and should be
  the documented default.
- `replace_patterns: true` makes the user's `patterns` replace the built-ins entirely.
  This is the "override the configuration" escape hatch, and it is the one thing `retry`
  lacks that users will want here, because a provider changing its wording should not
  require living with a stale built-in.
- Scalars resolve by key presence so an explicit `0`/`false` beats the built-in.
- A provider with no user entry and no built-in yields `None` (feature off for that
  provider), exactly like `get_retry_config`.

Add the section to `src/sase/config/sase.schema.json` next to the existing `retry`
definition, with `additionalProperties: false` and descriptions on every field, matching
the style already used there.

### Plugin hook

Add to `src/sase/llm_provider/_hookspec.py`, mirroring `llm_default_retry_config`:

```python
@hookspec(firstresult=True)
def llm_default_usage_limit_config(self) -> ProviderUsageLimitConfig | None: ...
```

Aggregate it exactly like `retry_config._built_in_defaults()` does — iterate
`iter_plugins()`, tolerate a missing method, swallow per-plugin exceptions — so
third-party provider plugins stay compatible without implementing it. Also expose it
through `_registry_metadata.provider_metadata()` next to the existing `retry_config`
entry so `sase doctor`/provider metadata can show it.

### Built-in patterns per provider

Implement each provider's hook in its own module. These are the shipped defaults; they
must be exactly this evidence-driven.

`claude.py` — anchored on the confirmed template `You've hit your <label>`:

```python
patterns=[
    "you've hit your usage limit",
    "you've hit your weekly limit",
    "you've hit your session limit",
    "you've hit your opus limit",
    "you've hit your sonnet limit",
    "usage limit reached",
    "claude usage limit reached",
]
exclude_patterns=[
    "usage limit approaching",
    "grace window active",
    "approaching your",
    "fast limit reached",       # fast-mode cooldown, not an account limit
    "close to your usage limit",
]
```

`codex.py` — the literal family from the binary:

```python
patterns=[
    "you've hit your usage limit",
    "usage limit reached",
    "to get more access now, send a request to your admin",
]
exclude_patterns=["usage limit approaching"]
```

`grok.py`:

```python
patterns=[
    "reached your free grok build usage limit",
    "usage limit for now",
    "get supergrok for much higher limits",
]
```

`qwen.py` and `agy.py` — transport/quota level, since neither ships distinctive prose:

```python
patterns=[
    "resource_exhausted",
    "quota exceeded",
    "insufficient_quota",
    "you exceeded your current quota",
]
```

`muse.py` and `opencode.py` — no evidence was found in their shipped artifacts. Give
them a conservative shared baseline (`"usage limit reached"`, `"quota exceeded"`,
`"insufficient_quota"`) and say so in a code comment, so a future agent knows these are
unverified and why.

`fakey.py` — ship a deterministic trigger (`"FAKEY-USAGE-LIMIT"`) so the `enforce` and
`notify` phases can be tested end to end, exactly as `fakey` already does for retry.

### Matching semantics

```python
def normalize_for_match(text: str) -> str
```

- Apply `unicodedata.normalize("NFKC", text)`.
- Translate U+2019, U+2018, U+02BC, U+00B4 and the backtick to ASCII `'`.
- Casefold.
- Collapse runs of whitespace to a single space.

Then `is_usage_limit_error(text, config)` returns True when any `patterns` entry is a
substring of the normalized text **and** no `exclude_patterns` entry is. Patterns are
normalized with the same function at match time so a user can write either apostrophe
and get the same result.

Exclusions are checked against the whole text, not per-pattern: if Claude's grace-window
advisory is anywhere in the captured output, this is not a disable-worthy failure.

### Reset-hint parsing

```python
def parse_reset_hint(text: str, *, now: float) -> tuple[float | None, str | None]
```

Handle the forms actually observed, all case-insensitively, on the normalized text:

- `resets <h>[:<mm>]<am|pm> (<IANA zone>)` — the confirmed Claude form, e.g.
  `resets 8pm (America/New_York)`. Resolve in the named zone via `zoneinfo`; if the
  resulting instant is in the past, roll forward one day.
- `resets <h>[:<mm>]<am|pm>` with no zone — resolve in the local timezone via
  `sase.core.time.get_timezone()`, same roll-forward rule.
- `resets in <N><unit>` / `try again in <N><unit>` with `h|hr|hour|m|min|minute` units,
  including compound `2h 15m`.

Rules that keep this safe:

- Add a 60-second grace buffer so sase never re-enables a hair before the provider does.
- Clamp the final duration into `[min_disable_seconds, max_disable_seconds]`.
- Any parse failure, ambiguity, or `honor_reset_hint: false` falls back to the resolved
  `disable_seconds`. Parsing is an optimization and must never be the reason a disable
  does not happen.

### Tests for this phase

Add `tests/test_llm_provider_usage_limit_config.py` and
`tests/test_llm_provider_usage_limit_defaults.py`, following the existing
`test_llm_provider_retry_config.py` / `..._retry_defaults.py` structure.

Include a regression corpus containing the **verbatim** captured strings, apostrophes
intact:

- `"You've hit your weekly limit · resets 8pm (America/New_York)"` → matches `claude`,
  and parses a reset hint.
- `"Error: You’ve reached your free Grok Build usage limit for now. Get SuperGrok for much higher limits, or try again later: https://grok.com/supergrok?referrer=grok-build"`
  (U+2019) → matches `grok`.
- `"You've hit your usage limit. Upgrade to Pro (https://chatgpt.com/explore/pro), visit https://chatgpt.com/codex/settings/usage to purchase more credits"`
  → matches `codex`.

And explicit negative cases that must NOT match:

- `"[Usage limit approaching. Checkpoint now: finish the current step…]"`
- `"[Usage limit reached — grace window active. Wrap up: finish or checkpoint…]"`
- `"Fast limit reached and temporarily disabled · resets in 5m"`
- `"You're close to your usage limit"`

Cover merge semantics (additive, `replace_patterns`, falsy-scalar override),
normalization (both apostrophes, NFKC, whitespace), reset parsing (each form,
roll-forward, clamping, failure fallback), and that every registered provider except
intentional abstainers returns a built-in config.

---

## Runtime disable and retry precedence

Phase `enforce`. Depends on `detect`.

### Hook point

`src/sase/llm_provider/_invoke.py` is the correct and only place to detect, because it
is where the provider identity and the failure text are both exact. Use
`execution_provider_label`, not `requested_provider_label`: after load-balancing or an
execution override, the provider that actually ran and actually hit the limit is the
execution one, and disabling the requested label would disable the wrong provider.

In the `except subprocess.CalledProcessError` handler, after `error_content` is
assembled and before `raise LLMInvocationError(error_content)`, call a new entry point:

```python
handle_possible_usage_limit(
    provider=execution_provider_label,
    error_text=error_content,
    model=context.metadata_model,
    artifacts_dir=artifacts_dir,
)
```

Apply the same call in the `except LLMInvocationError` handler, since providers that
fail through a parsed stream rather than a non-zero exit reach that path.

Do **not** call it from the generic `except Exception` handler: an arbitrary internal
error is not provider-attributable evidence of a usage limit.

Detection must only ever run on these failure paths. That, plus `exclude_patterns`, is
what prevents Claude's advisory "approaching" text — which appears in _successful_ runs
— from disabling anything.

### Writing the disable

Add `src/sase/llm_provider/usage_limit_disable.py` (or place it beside the config
module; keep `_invoke.py` thin) implementing `handle_possible_usage_limit`, which must:

1. Return immediately when `enabled` is false, when the provider has no resolved config,
   or when the text does not match.
2. Check `get_active_provider_disable(provider)` first. If a disable is already active,
   do nothing further — no extension, no second notification. Many agents run
   concurrently and will all fail within the same minute; the first one wins, and the
   rest must be silent. Log at debug level for traceability.
3. Write the disable through the existing Rust-backed API:
   `disable_provider_until(provider, expires_at, source=…)` when a reset hint was
   parsed, otherwise `disable_provider(provider, seconds, source=…)`.
4. Use a structured, stable `source` value of `"usage_limit"`. The Rust store validates
   only that `source` is non-empty, so no wire-schema change is needed; the `surface`
   phase renders this value.
5. Return the `UsageLimitDetection` so callers and tests can assert on it.

Wrap the whole body in `try/except Exception` and log rather than raise. The provider
error must always propagate unchanged; this feature is strictly additive to the failure
path.

The call is also the natural place to fire the `notify` phase's sender, guarded by the
resolved `notify` setting.

### Retry precedence

This is the behavioral fix that makes the feature worth having.

`src/sase/axe/run_agent_exec_retry.py::handle_workflow_error` currently classifies the
error against retry patterns and, for `codex`, would treat a usage-limit failure as
retryable and sleep through `[60, 300, 1800]`.

Change the ordering so a usage-limit failure is checked **first**:

- If the failing error is a usage-limit error for the provider that ran, do not take the
  wait-and-retry branch for that same provider. Sleeping 30 minutes to re-hit a weekly
  limit helps nobody and holds a workspace claim hostage.
- Still allow the `fallback_model` branch when the configured fallback resolves to a
  _different_ provider that is not currently disabled — routing already consults
  provider disables, so this degrades gracefully to a working provider instead of
  failing outright. If the fallback resolves to the same disabled provider, skip it and
  raise.
- Record the reason on the attempt snapshot (`snapshot_attempt`) so the ACE attempts
  view explains why no retry was attempted.

Cover this with tests in `tests/test_axe_run_agent_exec_retry.py` (or a new sibling): a
`codex` usage-limit error must not consume `wait_times`, and a transient `429` that is
_not_ a usage-limit match must still retry exactly as it does today. That second test is
the regression guard for the pattern overlap.

### Telemetry

Add a counter beside `LLM_RETRIES` in `sase.telemetry.metrics`, e.g.
`LLM_PROVIDER_AUTO_DISABLES` labeled by provider, incremented once per disable actually
written.

---

## Rich usage-limit notification

Phase `notify`. Depends on `enforce`.

Add `notify_provider_usage_limit_disabled(...)` to `src/sase/notifications/senders.py`,
following the shape of the existing `notify_axe_*` senders (build a `Notification`, call
`append_notification`, return the id).

The `Notification` model already supports everything needed: `icon`, `color`, `notes`,
`tags`, `action`, `action_data`.

Compose it from the `UsageLimitDetection` plus the agent context:

- `sender`: `"llm.usage_limit"`
- `icon`: a single glyph consistent with the repo's existing usage (`"⛔"`)
- `tags`: `["llm", "usage-limit", <provider>]`, run through
  `normalize_notification_tags`
- `notes`, in this order — this is the "what, how long, why" the request asks for:
  1. `Claude disabled for 8h 4m — weekly usage limit reached` (use the provider's
     **display name**, not the registry key, per the project's user-facing naming rule;
     resolve it through the provider metadata `display_name`.)
  2. `Re-enables at 8:00 PM EDT (Sat Aug 16)` when an expiry exists, or
     `Disabled until cleared` when it does not. When the expiry came from the provider's
     own reset hint, say so: `…as reported by the provider`.
  3. `Triggered by agent <agent_name> on <model>` when that context is available.
  4. `Provider said: "You've hit your weekly limit · resets 8pm (America/New_York)"` —
     the raw trigger text, truncated to a sane display length with the existing
     `truncate_error_snippet` helper.
  5. A routing note naming what happens next, e.g.
     `Launches now route to the next enabled provider in each alias.`
- `action`/`action_data`: point at the Models panel so the user can review or clear the
  disable in one keystroke. The notification action vocabulary is currently a small
  legacy set (`ViewErrorReport`, `JumpToMentorReview`, …) and the ACE app already
  exposes an `open_models_panel` action; wire a new notification action through the same
  dispatcher those use. If that wiring turns out to be larger than this phase, ship the
  notification without an action rather than inventing an unhandled action string, and
  note the gap.

Dedup: the `enforce` phase already guarantees one write per disable window by checking
for an active disable first, so the notification inherits that. Add a test that N
concurrent-style detections for the same provider produce exactly one notification.

Respect the `notify` setting so a user can keep the auto-disable but silence the
notification.

Tests belong beside the existing notification sender tests and should assert on the
composed notes, tags, and dedup rather than exact prose.

---

## Surface the disable reason and document the feature

Phase `surface`. Depends on `enforce`.

### Render provenance

`TemporaryProviderDisable.source` is currently written only as `"ace"` and is **not
rendered anywhere**. After this epic there are two meaningful values, and the difference
matters to the user: a disable they chose versus one sase chose for them.

- `src/sase/ace/tui/modals/models_panel_*`: in the provider rows/description rendering,
  show the provenance for an active disable — manual (`"ace"`) versus automatic
  (`"usage_limit"`), e.g. `disabled · usage limit` vs `disabled · manual`.
- `src/sase/ace/tui/widgets/provider_disables_indicator.py`: include the same
  distinction in the pill tooltip, which already formats remaining time.

Treat `source` as free-form: render known values specially and fall back to showing the
raw string for anything else, so a future writer or a third-party plugin cannot break
the panel.

### Documentation

- Add the fully commented `usage_limit:` block to `src/sase/default_config.yml` directly
  after the `retry:` block, with the built-in patterns shown commented-out as grammar
  examples (matching how `retry` documents `fakey`, and how `model_aliases` notes that
  shipped defaults live in code). State plainly that user `patterns` extend the
  built-ins and that `replace_patterns: true` overrides them.
- Update the user-facing docs under `docs/` wherever provider disables and the retry
  subsystem are described, including the reset-hint behavior and the 24-hour default.
- Note in both places that the disable is machine-wide and self-expiring, and that it
  can be cleared early from the ACE Models panel.

---

## Verification

Each phase runs `just check` before handing off. The combined tree runs
`just check-full` through `/sase_monitor` before landing, since this touches the
llm_provider, axe, notifications, and ACE trees together.

Beyond the per-phase tests above, the epic is done when a `fakey`-driven end-to-end
exercise shows: a failing invocation whose stderr contains the fakey trigger disables
only `fakey`, writes one notification naming the provider and duration, causes no retry
sleep, leaves other providers untouched, and lets the original provider error reach the
agent's error output unchanged.
