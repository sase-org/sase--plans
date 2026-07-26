- **PLAN:** [../202607/prompt_word_definitions_spellcheck.md](../prompt_word_definitions_spellcheck.md)

 Can you help me add support to the `K` (preview) keymap in the prompt
input widget for normal words when no other use-cases match?

- The dict_and_spell_cli_tools.md research file (in the research sidecar repo)
  contains research generated from previous agents regarding existing CLI tools
  that we can install that would allow us to support fetching the definition(s)
  for words from the command line and/or spell checking words from the command
  line. Review this research for inspiration before deciding on your solution.
- If you decide that we should install any of the CLI tools mentioned in the
  research file (I'm assuming we will install one of these tools for dictionary
  lookup and another for spell check), then the agent responsible for that
  should submit a sase gate notification to prompt me to approve the appropriate
  install command for this machine.
- When the `K` keymap is used on a normal word (i.e. a word not matched by any
  existing logic for this keymap) that is correctly spelled, we should display
  one or more definitions in a pop-up panel for that word.
- If the word is incorrectly spelled (i.e. not a word in our dictionary), we
  should show a new spellcheck panel with recommended spelling fixes that the
  user can select and apply with a single key press.
- Make sure we document any CLI tools that we install as optional CLI tools for
  sase and make sure that we update the `sase doctor` command accordingly (this
  should probably be a `--deep` check).
- I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful!

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
  