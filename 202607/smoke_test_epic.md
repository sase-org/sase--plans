---
tier: epic
title: Smoke test epic for phase sizes
goal: Test phase size routing, child epic naming, and recursive cascades
phases:
- id: small-phase
  title: Small phase test
  size: small
  depends_on: []
  description: Small phase - should route to @phase_worker
- id: medium-phase
  title: Medium phase test
  size: medium
  depends_on:
  - small-phase
  description: Medium phase - should get
- id: large-phase
  title: Large phase test
  size: large
  depends_on:
  - medium-phase
  description: Large phase - should get
bead_id: sase-81
---

# Smoke Test Epic for Phase Sizes

This is a test epic to validate phase sizing, child epic naming, and recursive cascades.

## Small Phase

No special routing, should use @phase_worker.

## Medium Phase

Should have #plan appended and use @phase_worker.

## Large Phase

Should have #plan appended and route to @smartest model.
