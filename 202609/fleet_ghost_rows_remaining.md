---
tier: epic
title: Finish fleet ghost-row read compatibility
goal:
  Owner-side fleet presentation, gateway version reporting, and the viewer's
  invalid-feed diagnostics agree with local behavior, and fresh Athena-to-Apollo
  acceptance shows no orphan family rows or hidden feed failures.
parent_bead: sase-xe.16.11.7.16
phases:
  - id: owner-presentation-parity
    title: Align owner presentation with local family history
    depends_on: []
    size: medium
    description:
      "owner-presentation-parity: suppress orphan terminal family members in the shared
      Rust presentation policy while preserving active and paginated families."
  - id: gateway-version-contract
    title: Publish the gateway version in fleet hello
    depends_on: []
    size: small
    description:
      "gateway-version-contract: add an honest read-compatible gateway service/version
      identity to the authenticated hello wire and its contract tests."
  - id: viewer-feed-honesty
    title: Complete viewer version and feed diagnostics
    depends_on:
      - gateway-version-contract
    size: medium
    description:
      "viewer-feed-honesty: consume the real gateway version and preserve reachable
      invalid/stale-host diagnostics through every projection path."
  - id: release-live-acceptance
    title: Adopt the fixes and complete live acceptance
    depends_on:
      - owner-presentation-parity
      - gateway-version-contract
      - viewer-feed-honesty
    size: medium
    description:
      "release-live-acceptance: release and adopt the Rust changes, redeploy both
      machines, capture clean live evidence, and complete the reopened acceptance beads."
proposed_by: bbugyi200.apollo.sase-xe.16.11.7.16.land
create_time: 2026-09-15 08:35:12
status: wip
---

- **PROMPT:**
  [prompts/202609/fleet_ghost_rows_remaining.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/fleet_ghost_rows_remaining.md)
- **PARENT:**
  [202609/fleet_ghost_rows_readcompat.md](https://github.com/sase-org/sase--plans/blob/main/202609/fleet_ghost_rows_readcompat.md)

# Plan: Finish fleet ghost-row read compatibility

## Context and boundaries

The landing audit found three concrete gaps in work previously reported complete:

- Apollo's fresh `include_terminal` fleet catalog still serves seven dead terminal
  `lane--gate` member records whose family root is absent. The viewer consequently
  synthesizes a `lane` family row, even though Apollo's local agent list is empty.
- `sase machine status` reads a fabricated `service_versions` hello field that the real
  `FleetHelloResponseWire` does not publish, so the promised gateway version/skew signal
  cannot work against a real gateway.
- Config diagnostics rebuild `FleetRowsProjection` without copying `host_feed_issues`,
  and a real invalid host has no row from which its full diagnostic can be opened. The
  existing detail test fabricates a row and therefore does not cover the live path.

The compatible-capability validator and stale-cache chrome already implemented by the
parent epic remain in place. Do not redo those pieces. The viewer's missing-family-root
materialization is also intentional for pagination and active families; do not delete it
to conceal bad owner data.

Post-start drift has been reviewed. The primary repository's test splits, notification
docs, relaunch-environment preservation, and sidecar quarantine changes do not overlap
this feature and must remain intact. The Rust capability fix has since shipped in
`sase-core` 0.34.29, while the primary lock still resolves 0.34.28. Treat 0.34.29 as the
baseline for the new Rust work and adopt the release containing all remaining core
changes once, rather than performing an intermediate dependency ratchet.

## Phase: Align owner presentation with local family history

Implement family-aware terminal selection in the shared Rust backend and gateway owner
read path. The served presentation catalog must follow the same semantic rule as local
listing: terminal nested family/gate member records are not standalone recent-history
rows; a completed family is represented by its root record. In particular, the observed
shape of seven dead `lane--gate` records with `parent_timestamp` set and no presentable
root must yield no `lane` catalog row.

Extend the presentation candidate facts and policy in `sase_core` as needed, and derive
those facts from the gateway's full indexed snapshot before pagination. Keep active,
unknown, waiting, and question-protected family members visible. Preserve legitimate
root rows and explicit history behavior. Keep client-side missing-root synthesis because
an active root can legitimately fall on another page; the owner must remove only records
that local presentation semantics would suppress.

Add focused Rust policy tests plus gateway read tests covering:

- orphan terminal `lane--gate` members matching the live Apollo record shape;
- a terminal family root with nested completed members;
- active/protected members whose root is outside the returned page;
- presentation versus explicit-history scope, dismissal lineage, age, and count bounds.

Run the focused suites and the repository's required verification for the Rust changes.

## Phase: Publish the gateway version in fleet hello

Add a non-secret service identity and package version to the authenticated fleet hello
response using the same authoritative package version already reported by the gateway's
health endpoint. Give the field an explicit gateway meaning (for example,
`sase-gateway`) rather than claiming that the gateway can attest to an unrelated host
CLI version.

Update the wire type, route construction, generated/declared contract surface, and route
serialization tests together. The change must be additive and readable across rolling
upgrades: older viewers continue to accept hello responses without the field, and older
gateways remain usable by the new viewer without inventing a version. Do not expose
credentials, build paths, or other host details.

Run the focused gateway/contract suites and the repository's required verification.

## Phase: Complete viewer version and feed diagnostics

Consume the actual hello service/version shape from the preceding phase. Human and JSON
`sase machine status` output must identify the remote gateway version and report skew
against the correct local compatible component; an absent field from an older gateway
must remain an explicit unknown, not a guessed match or a connection failure. Base tests
on a serialized real `FleetHelloResponseWire` fixture or equivalent shared contract
shape, not a Python-only invented payload.

Repair the projection-copy path in `_fleet_refresh.py` so `host_feed_issues` survives
when config diagnostics are appended. Then make a real zero-summary invalid-host result
selectable or otherwise provide an equally direct detail surface where the normalized
diagnostic code/message and cache age can be reached from the agents view. Do not create
a healthy-looking phantom agent row. Cover the real normalization path, config
diagnostics coexistence, invalid and stale cached hosts, selection/detail behavior, and
healthy-host absence of noisy chrome.

Route refreshes through the existing cached/background path, keep pump callbacks thin,
and use selective updates where the affected surface permits. Recheck cache keys so a
status or cache-age transition cannot reuse stale header, row, or detail text. Preserve
all public CLI compatibility and JSON fields already shipped by the parent epic.

Run focused dispatch, CLI, fleet-model, and TUI tests, then the primary repository's
required verification.

## Phase: Adopt the fixes and complete live acceptance

After the Rust phases land, publish the normal `sase-core` release containing both the
0.34.29 capability fix and the new presentation/version work. Ratchet the primary
dependency floor and lock through the supported release-adoption flow, update bindings
or adapters if the wire requires them, and exercise the cross-repository integration
tests. Do not add a Python fallback for the Rust presentation policy.

Install the resulting current SASE build on Athena and Apollo and restart Apollo's
gateway so the running service matches the installed build. Verify machine status in
both human and JSON modes reports the real gateway identity/version and gives an honest
match, skew, or unknown result.

With fresh, non-cached federation data, capture raw catalog and interactive ACE evidence
for project and by-machine grouping. Acceptance requires:

- Apollo's local list and its presentation catalog agree that the old `proj/lane` family
  has no presentable row, including with terminal presentation enabled;
- no `lane`, `attempt-0`, or `y--plan` ghost appears in either grouping;
- legitimate remote family/clan/host chips, human project labels, timing text, and
  liveness display match equivalent local rows;
- remote rows do not show local-only `here` state; and
- healthy feeds stay quiet while deliberately invalid or stale cached feeds retain the
  promised visible and detail-level diagnostics.

Register the live transcript/screenshots as a durable artifact and attach the reference
to `sase-xe.16.11.7.15.7`. Once the evidence meets every criterion, close that reopened
phase normally with the verified result, then close `sase-xe.16.11.7.16.1` normally as
the parent acceptance phase. Leave `sase-xe.16.11.7.15` and `sase-xe.16.11.7.16` open
for their waiting land agents.

Run the required full verification in both repositories after dependency adoption and
rerun Symvision where applicable before handing control back to the parent land agent.
