---
tier: epic
title: Plan-file rollout
goal: Exercise the host-owned plan-file launch.
phases:
- id: core
  title: Build the core
  depends_on: []
- id: cli
  title: Add the CLI
  depends_on:
  - core
- id: verify
  title: Verify the result
  depends_on:
  - core
  - cli
create_time: 2026-07-19 20:10:42
status: wip
---
# Plan

Execute the rollout.
