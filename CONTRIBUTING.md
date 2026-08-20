# Contributing

Fork, branch, open a pull request. There is no other path in, and that is
deliberate -- the project is maintained by reviewing changes from installs the
maintainer cannot see.

## What a change has to carry

**Evidence, not reasoning.** Every rule in `commands/bridge.md` is there because
something failed, and `docs/protocol.md` records what. A pull request that
changes behavior should say what happened on a real run: what you did, what the
protocol did, and what it should have done instead. "This would be cleaner" is
the one argument that will not move a rule.

Findings are welcome without a fix. A clear report of something that broke, or
of a rule that held only because you happened to ignore it, is more useful than
a patch that guesses at a cause.

## Before you propose a rule

This used to live in the command itself and was moved here: it governs people
amending the protocol, not sessions running a bridge, and a session should not
pay 33 lines of context for instructions about pull requests it will never
write.

**Name the two situations your rule cannot tell apart, and say whether they
differ in consequence.** If they do, it is broken however correct it reads on
its own. Repeatedly, amendments to this protocol have failed exactly this, each
in a different disguise, and none was catchable by its author -- which is why it
is a required step here rather than something review is trusted to catch. The
amendment that passed clean is the reason it is worth writing down: it wrote its
negative case unprompted, so this is learnable rather than a tax. The full
evidence table is in `docs/protocol.md` under "One defect, six times, in five
costumes".

The disguises, in the order they have actually appeared:

- **Absence.** A field that may be omitted makes an omission and the negative
  case identical to whoever reads it. Write `carrying: nothing`, or make the
  omission impossible.
- **Private judgement.** A trigger each party evaluates for itself is not a
  shared trigger. Two seats disagree about whether the rule fired and both
  produce correct-looking output. Key it to something on the shared surface.
- **Time.** A condition read from a growing file carries a timestamp whether or
  not anyone wrote one down. Ask what happens to a record written before the
  condition flipped.
- **Half a pair.** Enumerating the dividend and leaving the divisor a bare
  number. The half you did not name is where the miss hides, because a miss
  cannot appear in a list you never wrote.
- **A qualification that does not travel.** A key meaningful only once qualified
  needs qualifying everywhere it is used, not only where it is cited. This is
  the one that survives review, because the sentence stating it correctly is
  usually in the same document.

One default covers the cases this list does not: **fail toward the loud one.** A
false "needs attention" costs a line. A false "nothing here" costs the thing the
rule existed to prevent.

**Say what you did not test.** Every claim in this repo is scoped to the machine
it was observed on. If you changed a probe and ran it on exactly one
configuration, say so in the pull request. That sentence is not a weakness in a
submission here; leaving it out is.

## House rules

- **The command is the only copy of the protocol.** `commands/bridge.md` holds
  the rules; `docs/protocol.md` explains why they exist and never restates them.
  Three separate copies of this command have drifted apart already, twice inside
  a week. Do not add a fourth, in any form, including a quoted excerpt in a doc
  that will silently go stale.
- **No internal identifiers.** No employer or organisation names, no internal
  repo, host, share or ticket names, no paths that reveal somebody's layout. The
  evidence in `docs/protocol.md` is anonymized on purpose and reads perfectly
  well that way -- "four sessions spent fifty minutes on one log line" is the
  finding; naming the four adds nothing.
- **ASCII only.** No em-dashes, en-dashes, smart quotes or emoji. Use `--` and
  `->`. These files get round-tripped through PowerShell and shells that mangle
  everything else, and a stray dash has broken a parser here before.
- **Bump the version in both manifests**, `.claude-plugin/plugin.json` and
  `.claude-plugin/marketplace.json`. They must match, and the bump is how other
  installs learn there is anything to fetch -- including documentation fixes.

## Reviewing

Changes are verified on a second machine before merge wherever the claim allows
it. This has been worth doing every time: the last two merges each turned up a
further defect during verification that the original report could not have seen
from its own hardware.

If your finding is confirmed somewhere else, expect the merge commit to say so.
If it is not reproducible elsewhere, expect to be asked what is different about
your setup rather than to be turned down.
