---
plan: 202607/artifact_files_rename.md
---
 On the Agents tab in the TUI, in the sase context section of the agent metadata panel, we have a lane called Artifacts. That lane has a field within it that's also called Artifacts. Can you help me rename this field to Files? Make sure to update all references to this concept of Artifacts, which should be found widely across the codebase and possibly in other sase repos to Artifact Files to disambiguate from the more general term of artifacts (which is used to describe a variety of other concepts as well, like plan files and sase commits). Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 