---
status: done
tier: epic
title: Subscription capacity for Claude, Codex, and Grok
goal:
  Let subscription users inspect remaining provider allowances, their scope, resets, and
  freshness through an extensible shared backend, CLI, and responsive ACE experience.
phases:
  - id: capacity-domain
    title: Define the shared subscription capacity model
    size: medium
    depends_on: []
    description:
      "capacity-domain: Implement the Rust observation and public read contracts,
      validation, freshness, scope applicability, and summary rules specified in this
      plan, with deterministic fixtures and PyO3 coverage. No collection or UI."
  - id: capacity-store
    title: Persist observations and fence stale writers
    size: medium
    depends_on:
      - capacity-domain
    description:
      "capacity-store: Add the machine-local Rust store, account-generation fencing,
      complete-versus-partial merges, bounded atomic persistence, read-only loading, and
      thin Python facade. Coordinate the required core wheel and SASE dependency."
  - id: probe-runtime
    title: Add the provider extension and bounded probe runtime
    size: medium
    depends_on:
      - capacity-store
    description:
      "probe-runtime: Add optional dynamic usage hooks, typed probe contexts/results,
      process isolation and deadline handling, usage configuration, synthetic plugin
      fixtures, and the temporary epic beta flag. Preserve existing plugin behavior."
  - id: claude-usage
    title: Collect Claude subscription windows and passive updates
    size: medium
    depends_on:
      - probe-runtime
    description:
      "claude-usage: Implement the zero-inference Claude usage probe and supplementary
      rate_limit_event capture, account-mode detection, conservative date parsing,
      per-window merges, and noninterference tests for normal stream parsing."
  - id: codex-usage
    title: Collect Codex subscription windows through app-server
    size: medium
    depends_on:
      - probe-runtime
    description:
      "codex-usage: Implement bounded app-server initialization, account/read and
      account/rateLimits/read collection, multi-bucket normalization, capability
      degradation, account fencing, and protocol/process-cleanup tests."
  - id: grok-usage
    title: Collect Grok subscription allowance through ACP
    size: medium
    depends_on:
      - probe-runtime
    description:
      "grok-usage: Implement the Grok Build ACP billing extension collector, native
      percentage and verified legacy decoding, account-wide scope, honest unsupported
      and not-applicable states, and bounded process cleanup with fixture coverage."
  - id: usage-refresh
    title: Supervise and coalesce refreshes across clients
    size: medium
    depends_on:
      - probe-runtime
    description:
      "usage-refresh: Implement the shared durable refresh service, per-provider
      admission and receipts, deadlines, cadence/backoff, scheduled collection, and
      best-effort limit/reset triggers. Verify overlapping requests and crash recovery."
  - id: usage-cli
    title: Expose cached usage and explicit refresh in the CLI
    size: medium
    depends_on:
      - usage-refresh
    description:
      "usage-cli: Add sase usage list and refresh, a stable JSON contract, plain/Rich
      presentation, provider filters, precise exit behavior, completion/help, and
      offline doctor diagnostics. Build the shared presentation helpers for ACE."
  - id: providers-usage-ui
    title: Add a read-only Usage view to the Providers home
    size: medium
    depends_on:
      - usage-cli
    description:
      "providers-usage-ui: Evolve the provider modal into a shared Providers home with
      Usage and Routing views, cached first paint, durable update observation, all
      allowance details, discoverable navigation, and keyboard/PNG regression tests."
  - id: usage-context
    title: Show scoped capacity hints where users choose providers
    size: medium
    depends_on:
      - providers-usage-ui
    description:
      "usage-context: Add model and alias-member capacity hints and merge quiet usage
      attention into the existing provider indicator. Preserve routing, advisories,
      selection behavior, and responsiveness; verify narrow and monochrome layouts."
  - id: usage-release
    title: Verify the combined feature and remove epic scaffolding
    size: medium
    depends_on:
      - claude-usage
      - codex-usage
      - grok-usage
      - usage-context
    description:
      "usage-release: Integrate all collectors and surfaces, run adversarial end-to-end
      and performance checks, finalize documentation and compatibility evidence, remove
      the temporary beta flag, and pass monitored full verification before landing."
proposed_by: bbugyi200.athena.052
bead_id: sase-y5
create_time: 2026-09-09 19:52:50
---

- **PROMPT:**
  [prompts/202609/subscription_capacity.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/subscription_capacity.md)
- **BEAD:**
  [sase-y5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-y5/README.md)

# Subscription capacity

## Outcome and scope

Answer four questions: what percentage of an included allowance remains, which
models/products share it, when that particular window resets, and when SASE last
observed it. For example, Claude can have 80% of its session allowance left while a
model-specific weekly allowance has only 6% left. Both facts must remain visible.

Ship support for the existing `claude`, `codex`, and `grok` provider plugins. A fourth
provider implements the same optional collection interface; it needs no provider-name
branch in the core, refresh service, CLI, or widgets. Old plugins remain usable without
implementing anything new.

This is an epic because the shared Rust contract and release, plugin collection, process
supervision, and interactive experience are separate implementation units. Authoring is
`xlarge` work; the explicitly specified implementation phases are `medium` direct work.
Provider phases and refresh-service work can proceed independently after the extension
contract lands. Their shared-file ownership is described below.

Included in v1: multiple allowance windows, remaining percentage, resets, native
provider warnings/rejections, freshness/completeness, CLI inspection and refresh,
periodic collection, Claude stream supplementation, and contextual ACE presentation.

Excluded: API input/output token accounting, historical token totals, monetary usage,
balances or credit inventories, billing dashboards, credit redemption, paid-usage
changes, forecasts, history/attribution, automatic routing/admission based on capacity,
aggregate pool percentages, multi-account management, browser scraping, direct vendor
HTTP with extracted credentials, and threshold notifications. Existing token artifacts,
retry behavior, provider disables/priorities, and drain notifications retain their
current meaning. Observed included exhaustion is not a claim that all work will stop.

## Research and repository grounding

Read these with `sase artifact read` when implementing; their audited reads supply the
proposal's research provenance:

- `research:202609/provider_subscription_usage/provider_subscription_usage.md`
- `research:202609/subscription_capacity_experience/subscription_capacity_experience.md`

The acquisition report contains same-day live evidence from Claude Code 2.1.263, Codex
CLI 0.153.4, and Grok Build 1.0.13. This planning turn reviewed that evidence; it did
not run new authenticated probes. Treat these as verified baselines, not a promise of
compatibility with every older or future version.

Use the acquisition report's CLI-owned authentication, Rust store, optional hook, and
passive Claude source. Use the later experience report to resolve conflicts: display
percentage left; preserve scope and all windows; use one Providers home and the existing
attention indicator; make `refresh` wait by default; expose a versioned public read
model rather than the storage file. Do not implement the first report's permanent
three-provider ticker, arbitrary primary-window headline, raw-store JSON API, or
proposed later credit features.

Current official documentation also supports Codex account inspection through
`account/read` and separately identified limit buckets through
`account/rateLimits/read`; use account inspection instead of opening `auth.json`.
[Codex app-server](https://learn.chatgpt.com/docs/app-server#auth-endpoints) Claude
documents a JSON `claude auth status` command, useful for nonsecret account context, and
command-line process controls.
[Claude CLI reference](https://code.claude.com/docs/en/cli-reference) Grok documents a
weekly allowance shared across Grok products.
[Grok usage and limits](https://docs.x.ai/grok/faq#usage--limits) The precise Claude
print-mode usage format and Grok ACP extension remain empirical CLI contracts;
compatibility tests must preserve that distinction.

Relevant existing seams:

- `src/sase/llm_provider/_hookspec.py`, `_registry_plugins.py`, `registry.py`, and the
  `sase_llm` entry points in `pyproject.toml`: optional plugin capabilities.
  `registry.py` memoizes metadata, so observations must never enter that cache.
- `claude.py`, `codex.py`, `grok.py`, `_subprocess_claude.py`: vendor launch and stream
  integration. The Messages parser is also used by Claude-compatible CLIs; quota capture
  must explicitly apply only to the Claude runtime.
- `provider_disable.py`, `provider_disable_peek.py`, `usage_limit_disable.py`: existing
  reactive limit enforcement and display-cache precedents.
- `src/sase/procs/request.py`, `service.py`, `src/sase/ops/`, and
  `src/sase/ace/tui/durable_submit.py`: supervised argv-only operations. A concurrency
  key is not by itself a complete request-coalescing policy.
- `src/sase/default_config.yml`, `src/sase/chops/`, and `src/sase/scripts/`:
  configuration and AXE scheduled script integration.
- `models_panel_provider_modal.py`, `models_panel_providers.py`, `model_picker_rows.py`
  under `src/sase/ace/tui/modals/`, and `widgets/provider_disables_indicator.py`:
  existing provider presentation.
- `src/sase/main/parser.py`, `entry.py`, and `src/sase/doctor/checks_providers.py`:
  command registration/default-list notices and offline health checks.

Shared domain behavior belongs in `sase-core`, including validation, merging, freshness,
scope matching, attention classification, and refresh admission policy. Open it with
`sase repo open gh:sase-org/sase-core -r '<specific reason>'` and use only the returned
checkout. Start alongside `crates/sase_core/src/provider_disable.rs` and expose bindings
from `crates/sase_core_py`. Python owns plugin dispatch, vendor decoding and subprocess
effects, adapters, and presentation. Do not implement a Python domain fallback.

## Shared contracts and invariants

### Observation versus public view

Define typed, versioned structures in Rust and matching thin Python records. Keep
provider observations, persisted state, and public read views separate. Reject unknown
wire versions explicitly; public JSON additions may be additive within a version, but
changing meanings requires a new version. Missing data is null/absent, never zero.

The observation envelope contains provider ID, opaque collection-context ID and account
generation, observation ordering token/time, source, collection outcome, completeness
(`complete` or `partial`), plan/account mode when observed, and windows. Each window
contains:

- A stable provider-local key and vendor label. Codex keys include both bucket ID and
  window slot, so primary and secondary cannot overwrite each other.
- Native `used_percent`, finite and nonnegative; preserve values above 100.
- `resets_at` in UTC epoch seconds or null; optional positive duration and a period
  start only when supplied, never inferred as evidence of a fixed period.
- Structured applicability: provider/account-wide, product, explicit model IDs or model
  family mapping, or unknown. Preserve unknown vendor labels and identifiers.
- Its own `observed_at`, source, and vendor state (`allowed`, `warning`, `rejected`, or
  `unknown`). Numeric percentages must not synthesize vendor rejection.

Persist latest attempt/error separately from last successful full observation and each
window's observation. Use stable collection outcomes such as `ok`, `not_applicable`,
`unauthenticated`, `unsupported`, and `error`; reason codes distinguish `not_installed`,
`unsupported_cli_version`, `timeout`, `parse_error`, `account_context_changed`, and
similar actionable cases. A typed diagnostic must not contain raw stdout/stderr, account
IDs, emails, tokens, or filesystem secrets. An empty response is not `ok` unless the
protocol explicitly establishes an empty complete subscription inventory.

Bound provider/window counts and label/diagnostic lengths at the ingestion boundary.
Reject non-finite timestamps and observations implausibly ahead of the injected clock;
do not let a bad timestamp keep an allowance fresh indefinitely. Treat vendor labels as
plain text, strip terminal control sequences, and escape Rich markup on rendering.
Changing to an authoritative non-subscription/logged-out state removes current numeric
subscription conclusions even if historical observations are retained internally.

The public `schema_version: 1` envelope has `generated_at`, aggregate collection health,
and deterministically ordered providers. Each provider exposes an opaque
context/generation reference, collection status/reason, completeness, last attempt, last
full observation, windows, and a scoped summary. Windows expose original used
percentage, computed remaining percentage, freshness, reset/age, and vendor state. The
summary references its limiting window keys, reports scope and completeness, and is null
when there is no current quantitative conclusion. Return partial known constraints
separately rather than representing them as a complete minimum. Do not expose storage
paths, private account fingerprints, or proc internals in the ordinary usage view.
Refresh receipts are a separately typed operation envelope.

Store one measure, `used_percent`; derive `max(0, 100 - used_percent)` in core.
Presentation caps the meter and shows `exceeded by N percentage points` in detail.
Preserve raw precision in JSON. For text, positive sub-percent remainder reads
`<1% left`; only exact exhaustion reads `0% left`, and partial use cannot round to a
falsely exact `100% left`. Local low/very-low classification uses unrounded values.

### Freshness, membership, and ordering

Use injectable clocks. With cadence C=300 seconds, a window is fresh through 2C, stale
from >2C through 4C, and unknown after 4C. A passed reset immediately removes it from
current numeric summaries, regardless of age. Keep the historical reading in detail,
with `reset passed; awaiting observation`; never manufacture a new 100%. Missing reset
data is displayed as unknown and does not invent a reset schedule.

A failed attempt preserves previous observations with their original times plus an
`Update failed` diagnostic. A partial event changes only the named windows; it does not
refresh an absent model-specific window or the last-full-observation timestamp. A
complete result reconciles membership. Remove an absent window only if that full
inventory is newer than the window; retain newer stream observations. Track complete
inventory ordering/tombstones so an older event cannot resurrect a removed window.
Likewise, an older full probe cannot replace newer values or reset information. Use
request-start ordering for probes without vendor timestamps, not response arrival time;
retain receipt time independently. Define and test deterministic ties. Order attempt
outcomes as well as windows: an old timeout/error completing late must not replace the
collection status of a newer successful refresh in the same generation.

Decode as partial when recognizable windows coexist with malformed rows; record the
diagnostic and keep valid observations without using absence as deletion evidence.
Reject invalid individual quantities, duplicate/conflicting keys, impossible explicit
periods, NaN/infinity, and malformed resets. A known percentage with an unparseable
optional reset may remain partial with null reset; never guess a date.

### Account context

V1 follows the user's effective CLI login for each provider; it does not add account
selection or credential management. Resolve the same executable and provider-home
settings as invocation, honoring overrides such as `SASE_GROK_PATH` and `CODEX_HOME`.
Normalize SASE's temporary Codex shadow home to its real auth context; do not group
unrelated home overrides as one account.

Use vendor-provided nonsecret account/auth metadata when available, hash identity with a
private installation salt, and keep only the opaque fingerprint internally. Inspect
credential-file metadata only if necessary to detect change; never read their contents
or copy secret environment values into operation payloads, logs, or cache keys. API-key
presence alone does not establish the CLI's effective auth mode. Resolve it through the
provider interface. Changing mode, identity, or auth context invalidates the old context
and advances a generation atomically. Late results/events carrying an earlier generation
are discarded; logout/API mode must not retain a subscription bar. Generation comparison
applies even to terminal-error outcomes.

Account changes outside SASE can be detected only at an identity/metadata check; do not
promise instantaneous detection. Recheck around active probes and bind passive events to
the invocation's captured context. If a provider cannot establish identity continuity,
full probes replace its inventory without merging across invocations; skip unfenced
passive merges. Document this limitation and keep readings dated. An explicit context
mismatch shows unknown until a successful observation for that context. Do not claim a
metadata-only token rotation proves a new account; conservative cache invalidation is
acceptable when stronger evidence is unavailable.

### Applicability and summaries

For a concrete model, combine every known applicable account/product limit and every
matching model limit, then find the lowest remaining percentage. A model-specific window
supplements shared windows; it never replaces them. Provider plugins declare exact
mappings as part of observations/capability data. Unknown scope remains unknown; never
infer model applicability from bucket order, label substrings, or percent size. Unknown
potentially relevant scope makes the model conclusion partial.

The provider overview may show its lowest observed fresh window only with that window's
scope, e.g. `6% left · Model X week`, and expose the shared allowance beside it.
Stale/missing/unknown-scope inputs qualify the summary. Keep known rejected/low
constraints visible even when other data is unknown, dating stale vendor state. Do not
label the earliest reset among exhausted windows as service recovery time. Aliases/pools
show member observations and selector topology without a combined percentage or a
promise of the next selected member. Reading them must not consume round-robin state.

## Collection and refresh design

### Extension interface and process boundary

Add an optional static capability hook and dynamic
`llm_usage_probe(context) -> ProviderUsageObservation | None` hook to the existing
single-provider pluggy dispatch. Static capability discovery must do no I/O and may be
cached; actual observations must not be cached as metadata. Missing hooks mean
unsupported, with no changes to `LLMProvider.invoke` or `InvokeResult`.

The context supplies schema version, deadline, resolved executable/auth context,
generation and operation identity. Third-party plugins normalize vendor payloads to the
same domain contract. Catch unexpected plugin exceptions at the service boundary; do not
depend on plugin authors never raising. Add a documented passive-observation facade that
accepts the same fenced envelope for any future event-producing provider.

Run probes in isolated killable worker processes under the durable service; vendor
subprocesses are argv-only children. A timeout on a thread/future alone is insufficient.
Share only the bounded JSON-line transport/process machinery; keep JSON-RPC and ACP
handshakes in the individual collectors. Correlate response IDs, tolerate unrelated
notifications, bound line/total output and stderr, and handle EOF, malformed responses,
protocol errors, cancellation, and unsolicited requests. Never service login, token
refresh requests owned by a client, tools, or approval requests. Close stdin and
terminate/reap the owned process tree on every exit, including service termination. Do
not stop unrelated provider daemons or the user's active sessions.

Collect only through the installed, unmodified provider CLI under its own login. No
inference prompts, session/turn creation, direct web endpoints, or browser credentials.
Use a neutral non-project working directory and supported CLI options to suppress
workspace hooks, agents, MCP startup, session persistence, and self-update where
possible. Retain required auth/config context. Do not overwrite user configuration.
Never run normal SASE invocation/finalizers merely to obtain usage.

### Provider collectors

**Claude.** Use the research-verified `claude -p --output-format json "/usage"` path
after checking compatible CLI capability/version and effective subscription auth;
supplement with documented JSON `claude auth status` when needed. Isolate probe startup
from project/user execution hooks using verified supported controls. Check
zero-turn/zero-cost result markers and never substitute a natural-language prompt if the
command is unavailable. The implementation must establish a pre-inference guard on
supported versions; a post-response cost check alone is not a guarantee. If a new CLI
cannot establish that guarantee, return unsupported and retain passive collection,
rather than risking paid inference as a compatibility fallback.

Parse session, all-model weekly, and model-scoped weekly rows into stable keys. Cover
both `6:30pm` and `8pm`, explicit zones, time-only/date-bearing/year-bearing resets,
year rollover and DST; use the existing reset parser where it already supports a format,
or extend deterministic parsing in core. Anchor relative omissions to the observation
time. Ambiguous/localized wording degrades to partial/error; the recognized API-mode
message yields not-applicable. Do not silently count only the first two rows.

For actual Claude runs, recognize `rate_limit_event.rate_limit_info.unifiedWindows` in
`_subprocess_claude.py`; convert utilization fractions to percentage and preserve native
warning/rejection classification. Do not interpret omitted fields as zero. Pass events
through a bounded nonblocking buffer/coalescing sink with bounded shutdown flush;
persistence failure cannot stall or alter assistant output, tool/thinking records, token
artifacts, exceptions, or return codes. Tag events with the correct invocation
context/generation; do not capture similar event names from other runtimes.

**Codex.** Start `codex app-server` over stdio, perform `initialize` then `initialized`,
inspect `account/read`, and request `account/rateLimits/read`. Support the documented
multi-bucket response and legacy single bucket, optional windows/durations/names, and
native limit state. Request background reset-credit detail suppression only when
supported; never enable Reserve capabilities or consume credits. Do not use normal
`codex exec --json` token events as subscription capacity.
[Codex rate-limit contract](https://learn.chatgpt.com/docs/app-server#6-rate-limits-chatgpt)
API-only/other non-subscription auth is not-applicable; absence of login is
unauthenticated. Unknown auth or removed methods yield a precise unsupported/unknown
reason rather than a fabricated allowance. Ignore all token-activity/billing fields.

**Grok.** Verify the executable is Grok Build, using the existing identity check; honor
`SASE_GROK_PATH` and disable self-update. Start `grok agent stdio`, perform ACP
initialization, and invoke `_x.ai/billing` with the underscore wire prefix. The research
observed `config.creditUsagePercent` and `currentPeriod.end`; normalize the period and
explicitly label the shared account-wide scope. Support a legacy `monthlyLimit`/`used`
ratio only when sanitized fixtures establish both fields measure the same included
subscription allowance and the denominator is positive. Preserve its actual period; do
not label a monthly result weekly. Ignore monetary limits, on-demand/prepaid values,
history, and token totals. Missing method means unsupported CLI version; absence of
billing alone must not be mistaken for API mode. Use explicit auth/eligibility evidence
for not-applicable, and label unresolved cases honestly.

### Store, admission, and scheduling

Use `$SASE_HOME/llm_provider_usage.json` with an independent lock and atomic
replacement, private file permissions, schema checks, and bounded lock acquisition.
Separate bad provider records from healthy ones; report corruption rather than silently
returning an apparently healthy empty cache. Reads never repair/write. Core owns all
state transitions, including generation CAS, refresh reservations, due/backoff
decisions, and stale-writer rejection. Python owns executing admitted work through the
proc API.

One `submit_usage_refresh` service serves CLI, ACE, AXE, and limit-event triggers.
Reserve/coalesce at provider + normalized auth context. Joining one in-flight provider
must not lose other requested providers: return a batch receipt referencing existing and
new child operations and a result for every requested provider. Use expiring
generation-fenced leases/reservations and the existing proc concurrency mechanisms; test
replay, stale reservations, restarted supervisors, and overlapping subsets. An explicit
refresh after completed work starts a new attempt, not a permanent replay.

Use at most three concurrent provider probes initially, a 10-second end-to-end budget
per provider including auth/handshake, and a 30-second whole-batch deadline with
reserved cleanup time. Larger future provider sets remain bounded and report providers
that could not run within the batch. Persist/emit sanitized partial results as each
provider finishes. Joining or cancelling a waiter must not terminate a shared refresh
still owned by another client; stopping the owning proc follows existing proc semantics.

Configuration under `llm_provider.usage_metrics`:

- `enabled: true` is a permanent user preference after release; false stops probes,
  passive writes, scheduled requests and attention, while inspection can explain it.
- `refresh_seconds: 300`, with a validated minimum of 60 seconds.
- `warn_percent: 75` and `critical_percent: 90` refer explicitly to percentage used;
  validate `0 <= warn_percent < critical_percent <= 100`. UI copy uses percentage left.
- `providers.<registered-name>.enabled` provides optional per-provider overrides. Do not
  hard-code the initial three providers into the generic configuration schema.

Background eligibility is capability + installed CLI + reference from configured
provider/model aliases/defaults or explicit per-provider enablement; omit hidden test
providers. Explicit filters may inspect any registered provider and report unsupported
or configuration-disabled status, without overriding a user's collection opt-out.
Routing-disabled providers still refresh, since reset information is useful for them.

AXE gets a short scheduled chop that checks due work and submits durable refreshes, not
one poller per agent/project. Effective machine collection settings come from the
home/global configuration; project selectors may establish interest, but cannot race to
rewrite machine-wide cadence or account ownership. ACE requests due work after first
paint and while open as a fallback when AXE is absent; a normal tick never probes
inline. The same admission rule prevents duplicate polls across all callers. With
neither AXE nor ACE running, cached CLI inspection plus manual refresh still works.

Bound automatic retries with cadence-based exponential backoff capped at 30 minutes,
respect a supplied retry-after floor, and reset on success/context change. An explicit
refresh may bypass age/backoff from transient errors but joins in-flight work and
respects retry-after and a short abuse-prevention cooldown. Limit failures, observed
window resets, and disable expiry can mark a provider due once per event/cycle; they
never extend/clear a routing disable or generate a second limit notification.
First-paint, event storms, and passive-only updates must not starve a periodic full
inventory refresh. Subscription telemetry always fails independently of an agent run.

## CLI and ACE experience

### CLI contract

Add `sase usage list` and `sase usage refresh`, title `Subscription usage`; bare
`sase usage` follows the central default-list convention. List is strictly cached and
works without AXE. It shows all windows for eligible providers; absent data says
`No observations yet; run sase usage refresh`.

Both commands accept repeatable `-p/--provider`, `-j/--json`, `-P/--plain`, and
`-v/--verbose`. Refresh additionally accepts `-b/--background`. Refresh waits by default
using the same durable operation as ACE; progress belongs on stderr and finished data on
stdout. Background mode emits operation IDs or a typed JSON receipt. Reject incompatible
JSON/plain formats. Sort help commands/options and provide useful examples; required
values are positional, with every public long option having a short alias. Add parser
registration, dispatch, completions, and central default-list coverage. `--json` stdout
contains one JSON document and no progress/default notice.

Plain/non-TTY/`TERM=dumb` output is undecorated ASCII, one untruncated record per window
or provider state; honor `NO_COLOR`. Rich output and ACE share pure display helpers for
labels, meters, source/age and exact reset date/time with timezone. Domain decisions
come from core, not duplicate Python computations.

Exit rules: list returns 0 for reportable absent/stale/provider-error states, 1 for
unreadable/corrupt store, and 2 for bad invocation/provider names. Refresh returns 0
when requested applicable collectors succeed, with explicit not-applicable/unsupported
outcomes as data; return 1 for any failed/unauthenticated applicable request, partial
decode, timeout, or persistence failure, preserving healthy results; 2 for invalid
invocation. Reaching quota itself is not a command failure. Background exit 0 means
successful submission/attachment only. Global/per-provider collection-disabled refreshes
produce an explicit disabled outcome, never silently enable collection.

Add an offline `llm.usage` doctor check for capability/config, cache integrity, last
observed mode, age, and sanitized collector diagnostics. Normal doctor runs must not
launch provider calls or claim present auth validity from cached evidence. Point users
to `sase usage refresh` and provider-owned login/update instructions.

### Providers home

Evolve the existing provider modal into `Providers` with explicit `Usage` and `Routing`
views sharing selection/order. Existing routing entry points open Routing directly and
preserve their established controls. New usage entry points open Usage. In Usage, Enter
focuses/opens details, `u` requests Update usage, Esc closes, and Tab navigates focus.
Disable/prioritize/enable/drain actions exist only in Routing; the existing
Enter-to-disable binding must never fire in Usage.

Provide a persistent Launch Control/Models entry and a command-palette action searchable
by usage, quota, limits, and capacity. No new global chord is required. Update
`default_config.yml` and help content for local actions and entry points.

Show one scoped summary row per provider and every window in scrollable detail, with
remaining meter, scope, reset countdown, observation age, and separate routing state.
Never impose a three-window cap. Exact times, source, plan, and diagnostics belong in
details. Do not show billing balances. Known rejection is distinct from zero included
allowance and from a SASE disable. All labels must work without color.

At 120 columns use table plus detail; at 80 remove meters before semantic fields; at 60
stack scope/reset/age. Unknown gets a question mark and no numeric meter. Stale readings
say `Last observed ...`; after reset say `reset passed`, never a negative countdown. An
update preserves selection/scroll, values and their age, shows Updating separately, and
ends with one result summary including failures. Closing the view leaves the durable
operation running; reopening attaches to it.

Render only in-memory snapshots. Disk reads, parsing, core file locks, process
submission and result loading happen in workers, off both the event loop and serial
message pump. Reuse `spawn_pump_free_task` and the established refresh/coalescing paths,
cancel local tasks on teardown, and recapture selected identity after awaits. Ticks
compute freshness/countdowns from loaded facts and update only affected rows; they never
synchronously stat/parse the new cache in rendering or keypress handlers.

### Context and attention

Use model-picker advisory fields for scoped low/unknown capacity and preserve existing
model advisories alongside them. Shared constraints belong in provider headings;
specific constraints stay on affected rows. Alias detail lists members and topology,
without invoking the stateful selector or combining percentages. Never reorder, disable,
or reroute from capacity observations.

Extend `ProviderDisablesIndicator` with quiet attention for eligible providers while
preserving existing disable/priority state. No new always-on ticker. Show the highest
attention usage item plus a count with stable provider-ID tie-breaking; keep scope in
detail/tooltip. Known rejection outranks low, then unresolved collection problems; fresh
healthy state does not alert. Unused unsupported plugins do not create noise. Never let
an existing priority pill suppress rejection/usage attention. At narrow widths preserve
routing text plus an attention count. Usage clicks open the selected provider's Usage
view, and all details are also reachable by keyboard.

## Phase execution and acceptance

1. **capacity-domain** owns new core types/pure rules and initial PyO3 exports. Publish
   sanitized shared fixtures covering 0, fractional, 100 and >100 used; shared plus
   model-specific limits; unknown scope; mixed ages; reset expiry; and deterministic
   ties. Validate public JSON against expected semantics, not merely a round-trip of the
   implementation. Do not add provider-name switches.
2. **capacity-store** owns persistence, ordering/tombstones, account generations,
   refresh reservation primitives, and `src/sase/llm_provider/usage.py` (or a small
   package). Test concurrent full/partial writers, stale full deletion, resurrection,
   account switch during probe, lock timeout, corruption isolation, interrupted atomic
   writes, read-only behavior, permissions, and `$SASE_HOME` isolation. Core's
   release-plz owns versions; do not manually bump Cargo versions. Land/release the
   required wheel before raising SASE's current `sase-core-rs>=0.32.34,<0.33.0`
   requirement to the actually available compatible release. Update dependency/lock and
   binding capability checks together; test missing required exports explicitly.
3. **probe-runtime** owns hooks, generic process transport, config parsing/defaults,
   extension documentation and synthetic plugin fixtures. Demonstrate adding a test
   fourth provider without core/CLI changes and that old plugins still invoke normally.
   Include hanging plugin, oversized output, secret-canary exceptions/stderr, and
   descendant cleanup fixtures. Create temporary `provider_usage_metrics` beta
   scaffolding with `sase flag new` only if phases expose unfinished paths as they land,
   as expected here; read `sase_flags.md` and test both states. The durable `enabled`
   preference is separate from this temporary flag.
4. **claude-usage** owns a dedicated Claude collector module and only the necessary
   `claude.py`/stream-parser hook additions. Cover recorded prose variants, API/logged
   out states, malformed/localized rows, event partial merges and ordering, probe guard
   behavior, and unchanged normal/Claude-compatible stream outputs. Keep source/version
   notes beside redacted fixtures. No generic runtime edits unless an interface defect
   is coordinated with its owner.
5. **codex-usage** owns a dedicated Codex collector and `codex.py` hook. Use scripted
   app-server transcripts to prove handshake, method-only traffic without turns,
   response correlation, legacy/multi-bucket/null cases, unknown buckets, auth-mode
   changes, wrong/unsupported methods, deadlines, and real subprocess reaping.
6. **grok-usage** owns a dedicated Grok collector and `grok.py` hook. Cover Build
   identity mismatch, exact ACP extension wire name, weekly native and verified legacy
   shapes, missing period/percentage, API/free/enterprise evidence, malformed data, auth
   ambiguity, and absence of orphan children/sockets owned by the probe.
7. **usage-refresh** owns proc operation registration/runner, shared submit/join
   service, AXE scheduling and best-effort event triggers. Use the synthetic provider
   while real collectors develop. Test CLI/ACE/AXE overlap with different subsets,
   multiple accounts/contexts never merging, cooldown/backoff and reset storms,
   cancellation, stale leases, restart, future >3-provider batches, global opt-out, and
   partial success without blocking normal limit handling. No per-agent poller.
8. **usage-cli** owns parser/handler/completion, shared pure display helpers, doctor,
   and initial user/plugin docs. Test stdout/stderr and exit rules, corrupt cache,
   non-TTY/no-color, repeatable filters and unsupported providers, partial refresh,
   typed receipts, deadline behavior, and strict no-probe `list`/doctor. It can be
   developed from shared fixtures before all vendor collectors finish.
9. **providers-usage-ui** owns Providers views and discoverability. Add interaction
   tests for Enter/Tab/Esc, routing-view compatibility, update/close/reopen,
   preservation of selection during results, and failure states. Use shared fixtures for
   PNGs at 120/80/60 columns, including all windows and colorless meaning.
10. **usage-context** owns picker/alias hints and indicator integration. Cover shared
    rejection plus healthy model-specific allowance, a low model window that does not
    implicate other models, unknown bucket applicability, coexisting advisories, and
    every existing top-bar indicator together at narrow widths. Assert alias inspection
    cannot advance round-robin counters or change routing eligibility.
11. **usage-release** owns the integrated acceptance run, documentation reconciliation,
    and flag removal. Remove the Off branch, registry entry and flag bead using the flag
    lifecycle; retain permanent enable/cadence preferences. Verify all three real
    adapters through the actual CLI and ACE paths using deterministic fake CLI
    executables. A bounded manual smoke against installed authenticated CLIs may
    corroborate compatibility, but must never start an inference turn or put private
    readings in fixtures. Record what was actually verified and any unsupported version
    honestly. This phase depends transitively on all infrastructure and explicitly on
    every provider collector, so no placeholder collector can ship.

Across phases, read the relevant `sase_memory_read` notes before CLI, TUI, feature-flag,
or verification work. New memory edits and unrelated task filing are not part of this
plan. Keep shared module edits with their phase owner; provider phases own their
collector modules and registrations, preventing parallel merge contention.

## Final verification and release criteria

Run required local checks after each implementing phase: `just install` when the
ephemeral environment needs current dependencies, then SASE `just check`. Core phases
run core `just check`, including PyO3 with Python >=3.12; `cargo test -p sase_core`
alone is insufficient. Follow each checkout's instructions and open linked checkouts
through `sase repo`. Do not hand-edit generated core release versions.

The combined feature must pass fixture-driven tests without network, real credentials,
or vendor CLIs in CI. Add end-to-end scripts that hang, flood output, interleave
notifications, switch accounts mid-refresh, report unsupported methods, and die before
writing results. Secret-canary tests cover cache, JSON, stderr, logs, proc request and
result files. Assert refresh succeeds independently per provider and cleans up all owned
workers. Assert usage reads/refreshes cannot write provider disable/priority state,
route differently, redeem credits, create a model turn, or change billing.

Run targeted PNG tests and inspect actual/expected/diff artifacts before accepting
intentional goldens. Exercise a slow/hanging collector while navigating and opening/
closing Usage; record first-paint and key-to-paint measurements using existing TUI
instrumentation, targeting p95 <16ms. Confirm idle ticks do no new provider work and no
repeated full-store parsing, and observation updates do not rebuild the agent list.

Finalize docs in `docs/llms.md`, `docs/ace.md`, `docs/agent_providers.md` and the
provider extension guide, plus configuration/help examples. Explain included allowance
versus API statistics, scope and activity outside SASE, dated observations versus
guarantees, unsupported/auth failures, collection opt-out, account-context limitations,
and how to add a provider. Avoid turning technical diagnostics into ordinary product
copy.

Before landing the combined epic, run SASE `just check-full` only through
`/sase_monitor` with the required TESTING/TESTED statuses, plus applicable core and
visual checks. Keep the same strict no-fallback Rust dependency boundary. The host owns
commits/releases/landing according to the normal finalizer workflow.

Done means Claude, Codex and Grok each expose honest subscription states through the
generic interface; CLI and ACE agree on scope, remaining percentage and freshness;
failed/partial/stale observations never appear as full allowance; refresh remains
bounded and leaves agent execution responsive; all documented checks pass; and no
temporary beta branch remains. Longer-term usability observations can inform future
forecast work but are not a hidden requirement to wait a weekly cycle to ship v1.
