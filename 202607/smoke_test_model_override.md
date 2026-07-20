---
tier: epic
title: Smoke test with model override
goal: Test that explicit phase model overrides size-based routing
phases:
- id: large-but-explicit
  title: Large phase with explicit model
  size: large
  model: claude/opus
  depends_on: []
  description: Large phase but with explicit claude/opus model - should ignore @smartest
---

# Model Override Test

Large phase with explicit model should override size-based routing.
