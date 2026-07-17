---
plan: 202607/gate_option_queries.md
---
The user sent an image via Telegram with the following caption:

 it looks like we use too many plan approval telegram buttons for plans
with a tale tier. Can you help me make this a little clearer, but do so in a way
that generalizes sase gates (and the /sase_gate skill)?

- sase gate options (which translate to the buttons that Telegram should offer)
  should be constructed from the sase gate command query, which should support
  the AND and operators. The command query for plan files with a tier of tail is
  shown below but I use placeholders instead of the actual commands. Keep in
  mind that each AND is meant to be represented as an option (which defaults to
  being set). In other words, this is a way to express that the user can select
  one or more of the options being ANDed:

```
(approve AND commit) OR reject OR feedback
```

- The "Approve with selected add-ons" button should be made configurable by sase
  gates. We should configure it to use `Approve` (with a custom icon--just use
  the icon that the current `Approve` button uses) for the button/option text.
  All of the buttons/options should be configurable in this way.
- Simplify all of the interfaces (both UI interfaces and programmatic
  interfaces) for sase gates as much as you can but remember that the ultimate
  goal is to make sase gates as powerful and customizable as possible.

I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful! Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 

The image has been saved to:
/home/bryan/.sase/telegram/images/20260717_224603_AgACAgEAAxkB.jpg Please read
the image file and respond to the user's request.