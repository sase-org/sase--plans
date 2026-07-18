---
plan: 202607/statistics_tab.md
---
 The new telemetry tab on the SASE admin center panel is a mess. None of those graphs mean anything to me or anyone. Can you help me rename this tab to Statistics?

- Rip out any bits of the telemetry infrastructure that aren't useful for debugging issues later.
- We should get rid of all graphs unless you can find a way to produce some with great data/numbers next to them.
- We should limit the data set to the below metrics that we care about.
- With that said you should allow the user to express various preset time ranges and a custom time range for which they can aggregate these metrics. 
- You should also provide a way for the user to toggle between various different and useful views. Again these can include graphs but only if there are concrete numbers behind them that the user can see.

The metrics (listed using free form text, not one metric per bullet):

- How many sase agents have been run?
- How many of them failed versus completed successfully?
- How many of them made commits and how many commits were made per agent?
- Which LLM / agent CLI providers were used? With what models and with what effort levels ?
- Which sase skills and sase memories and sase workspaces were used the most?
- How many agents proposed plans? How many of those plans were epic plan files versus tail plan files? How many of those plan files were approved? How many phases were in each one of the epic plan files that were proposed?
- What was the agent's runtime? Make sure you provide various ways to split this metrIc / group this metric up across agent tribes/clans/families as well as individual agents.
- How many agents ask questions? How many questions were asked at once?

I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful!  Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 