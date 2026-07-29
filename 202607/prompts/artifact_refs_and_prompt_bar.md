- **PLAN:** [../202607/artifact_refs_and_prompt_bar.md](../artifact_refs_and_prompt_bar.md)

 Can you help me complete the work associated with the
`Extend plans: into a kind-tagged artifact reference, following the sase-9z playbook`
and `Artifact refs in the prompt bar` sections (recommendations #2 and #8) in
the artifact_refs_and_inspector.md research repo file?

- Keep in mind that we should not support the `research:` artifact type (see the
  sase-as epic bead for context).
- Make sure that `@`-prefixed artifact references have excellent syntax
  highlighting in the prompt input widget (and in external editors, provided via
  our treesitter and/or LSP support).
- Also, make sure that both the prompt input widget and editors (via LSP
  support) have excellent completion support for artifact references.
- I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful!

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 