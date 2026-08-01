- **PLAN:** [../202608/artifact_persistence_sidecars.md](../artifact_persistence_sidecars.md)

 Can you help me add much better support for artifact persistence /
linkage in sidecar repos?

- To start, any artifact references that are detected in the prompt should be
  copied to the local .sase/artifacts/ directory (maybe use a hash of the file
  contents in the artifact file name or something to ensure multiple copies of
  the file same file are supported if that file had different contents for
  different prompts?).
- `@<type>:` style references are artifact references, but so are `@` file
  references.
- Let's move the .sase/home/ directory (where we currently copy
  `@~/path/to/some/file.ext` file references) to .sase/artifacts/home/.
- When an agent commits using the `sase commit` command, all artifacts
  corresponding with that agent's prompt should be copied to an artifacts/
  directory right next to the prompts/ directory containing the prompt.
- The prompt stored in the prompt markdown file should then be modified to link
  (using inline markdown links) to the relevant artifact files in the same repo.
- As a part of this change, the `<project>--agents` sidecar repo should be made
  the canonical (and only) location for prompt markdown files. These files
  should still link to/from their corresponding plan files (if they have
  one--the fact that not all agent commits correspond with plan files but all
  correspond with prompts is why we are making this change) like they do now.
  Make sure this change doesn't break the `sase validate` command.
- I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful!

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 