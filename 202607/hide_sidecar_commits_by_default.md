---
tier: tale
title: Hide sidecar commits by default
goal: 'The Artifacts Commits sub-tab omits every sidecar repository by default and
  lets users opt the complete sidecar set into the timeline with the canonical `sidecar:true`
  filter, without regressing live filtering or refresh behavior.

  '
create_time: 2026-07-20 13:12:28
status: wip
prompt: 202607/prompts/hide_sidecar_commits_by_default.md
---

# Plan: Hide sidecar commits by default

## Context and product contract

The commits pane currently asks the VCS-log service for primary and linked repositories by default, but the
linked-repository resolver already returns both ordinary linked repos and modern SDD sidecars. `vcs_log.resolve`
discards that distinction and labels every returned repo as `linked`; the older, singular SDD resolution path is the
only repo classified separately. As a result, the pane's `include_sdd = False` default and its `d` toggle do not hide
all configured sidecars, especially split `plans`, `research`, and custom sidecars.

Establish one coherent contract:

- An empty commits filter includes primary and ordinary linked repositories but excludes every repository whose resolved
  role is sidecar.
- `sidecar:true` includes all available sidecars in the selected project or all-project scope. This includes modern
  configured/default split sidecars and a materialized legacy separate SDD repository.
- `sidecar:false` is accepted and has the same behavior as the default; canonical query serialization omits the default
  false value. The property is singular, non-negatable, case-insensitive, and accepts only `true` or `false`.
- Existing `repo:`, author/date/text, limit, project-scope, refresh, and fetch semantics compose with the sidecar
  property. In particular, selecting a sidecar by `repo:` still requires `sidecar:true`.
- Keep the configured `d` commits action as a compatibility shortcut, but make it toggle the same filter value
  represented by `sidecar:true`; do not retain a second, divergent SDD-only state.

## Preserve sidecar identity in the VCS-log domain

Extend the frontend-neutral VCS-log repo model in `src/sase/vcs_log/models.py` to represent `sidecar` explicitly,
aligning it with the identity already provided by `ResolvedLinkedRepo.kind` and the repository inventory. Update
`src/sase/vcs_log/resolve.py` to carry that kind through instead of coercing it to `linked`, and normalize the legacy
separate-SDD result to the same sidecar classification. Keep primary-preference deduplication intact so a physical repo
that is also a registered project remains a primary repo in all-project scope.

Thread a sidecar-inclusion control through repository resolution and `run_vcs_log`. Skip sidecar candidates before
collection when it is false so the default commits view neither reads nor fetches hidden repositories. Preserve the
existing CLI compatibility surface: `sase vcs log --sdd` continues to be the CLI opt-in but maps to the complete sidecar
set, while `sase vcs list` continues to request all repos. Update model documentation, ordering, render/adaptation
branches, and focused VCS log/list tests so there is no lingering assumption that modern sidecars are ordinary linked
repos or that `sdd` is a separate repo kind.

This work remains in Python: repo resolution is explicitly owned by the frontend-neutral Python adapter, while the Rust
aggregator receives only repo labels and commit lists and therefore needs no wire or binding change.

## Add the `sidecar:` commit-filter property

Extend `CommitLogFilterValues` and the parser/serializer/completion helpers in `src/sase/vcs_log/filter_query.py` with
the boolean sidecar value. Add precise diagnostics for missing, duplicate, negated, or non-boolean uses; serialize only
the active opt-in as `sidecar:true`; and preserve canonical round trips. Teach `CommitFilterBar` to complete the
`sidecar:` key and its `true`/`false` values with a concise hint, without resolving repositories or doing any I/O on the
keystroke path.

Cover default parsing, valid boolean spellings, invalid forms and spans, canonical serialization/round trips, token
chips, and completion context at the pure-query and widget levels. Existing free-text, exclusion, quoting, and
unknown-key behavior must remain unchanged.

## Make collection, previews, and caching sidecar-aware

Remove the commits pane's independent `include_sdd` state and derive collection scope from the active
`CommitLogFilterValues.sidecar` value. Include that value in collection specs, authoritative cache keys, snapshot
breadth/coverage, and generation reconciliation so an in-flight sidecar-hidden result cannot land as an exact
sidecar-inclusive result.

Update local preview filtering to use `VcsLogResult.repos` metadata when filtering commits, repos, and remote states. A
broad cached result collected with sidecars can preview `sidecar:false` exactly; a default narrow snapshot cannot invent
sidecar rows for `sidecar:true`, so it should render the available preview as inexact and let the existing debounced
background collection obtain the complete result. Preserve selection, detail debouncing, last-request-wins generation
checks, bounded authoritative caching, and off-thread collection. No repository discovery, filesystem access, fetch, or
other slow work may move onto the filter typing or Textual message-pump path.

Make the legacy `d` action update the canonical filter value and use the same state-change/collection path as a
submitted query. Retain its action/config key for user configuration compatibility, while changing user-facing labels
from SDD-specific language to sidecar language.

## Present and document the default clearly

Update the commits header, filter chips, footer hints, command metadata, and the `?` help modal so they describe
sidecars rather than a separate SDD-only scope. The inactive view should make the hidden default understandable, while
an active opt-in visibly carries the canonical `sidecar:true` chip. Update any affected help-width assertions and PNG
snapshots, including a state that proves the filter bar offers `sidecar:true` and the resulting timeline includes a
sidecar commit.

## Verification

Add or adjust focused coverage for:

- VCS-log resolution in current-project, explicit-project, and all-project scopes, proving linked repos remain visible,
  all sidecar roles are hidden by default, the opt-in includes them, legacy SDD is classified consistently, and
  physical-path deduplication still behaves deterministically.
- Collector argument plumbing and CLI/list compatibility for the sidecar inclusion flag.
- Commit-filter parsing, completion, canonical chips, and boolean errors.
- Commits-pane startup, live preview, submit/dismiss restore, cache exactness, `repo:` composition, refresh/fetch calls,
  the compatibility toggle, and selection/detail stability across sidecar visibility changes.
- Help/footer rendering and the dedicated ACE visual snapshot suite for the changed commits states.

Before final handoff, run `just install`, the focused query/resolver/collector and commits-pane/widget tests, and
`just test-visual`. Inspect visual diffs and accept snapshot updates only for the intentional sidecar UI changes. Finish
with the repository-mandated `just check` so formatting, lint, typing, the full test suite, and snapshot validation all
pass.
