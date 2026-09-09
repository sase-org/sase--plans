---
tier: tale
goal:
  Ensure the README and published Markdown documentation accurately describe the current
  SASE product and cover important recent behavior.
create_time: 2026-09-09 19:53:10
status: wip
---

# Plan: Audit and refresh product documentation

Review `README.md` and all user-facing Markdown under `docs/`, including the MkDocs
navigation and internal links. Compare documented commands, configuration, workflows,
and feature descriptions with the current CLI and implementation, and inspect product
commits made after the most recent documentation-focused update for changes that readers
need to know about.

Update existing pages where the best documentation location already exists, and add a
focused page or README section only when the audit finds a material navigation or
coverage gap. Preserve historical blog posts and generated image notes unless they
contain an active link or factual problem that affects the published site.

Validate the result with documentation/link checks, a strict MkDocs build, targeted
command verification, and the repository-required `just install` followed by
`just check`. Report the important gaps found, files changed, and verification outcome.
