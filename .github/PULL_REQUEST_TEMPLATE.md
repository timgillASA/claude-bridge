<!--
Read CONTRIBUTING.md first. Delete any section that does not apply, but do not
delete "Not verified" -- an empty answer there is itself an answer.
-->

## What happened

What you ran, what the protocol did, and what it should have done instead.
A finding without a fix is welcome; describe it here and leave the rest short.

## Why it is worth changing

Especially if nothing actually broke. "The list was wrong and the model
compensated" is a real reason; say so plainly.

## The change

What you changed and where. If it touches `commands/bridge.md`: say which
existing rule this replaces or generalizes (or that it is genuinely new), give
the net line change to the command, and confirm the reasoning landed in
`docs/protocol.md` rather than in the command. Include the smallest transcript
excerpt that demonstrates the failure, anonymized.

## The collapse test

For any new or changed rule: **what two situations can it not tell apart, and
do they differ in consequence?** See "Before you propose a rule" in
CONTRIBUTING.md. Answer it here even when the answer is "none found" -- an
omitted section is indistinguishable from a skipped check.

## Not verified

The configurations you could not test on. One machine, one OS version, one shell
-- name the limit. Do not leave this empty.

## Checklist

- [ ] Version bumped in **both** `.claude-plugin/plugin.json` and
      `.claude-plugin/marketplace.json`, and they match
- [ ] ASCII only -- no em-dashes, smart quotes or emoji
- [ ] No internal identifiers: host names, share names, repo names, paths
- [ ] The protocol is still stated in exactly one place
- [ ] The collapse test above is answered, not deleted
- [ ] For behavior changes: one adversarial review by a seat that did not
      write the change, recorded before or on merge
