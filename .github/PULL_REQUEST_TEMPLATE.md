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

What you changed and where. If it touches `commands/bridge.md`, say which rule
and confirm the reasoning landed in `docs/protocol.md` rather than in the
command.

## Not verified

The configurations you could not test on. One machine, one OS version, one shell
-- name the limit. Do not leave this empty.

## Checklist

- [ ] Version bumped in **both** `.claude-plugin/plugin.json` and
      `.claude-plugin/marketplace.json`, and they match
- [ ] ASCII only -- no em-dashes, smart quotes or emoji
- [ ] No internal identifiers: host names, share names, repo names, paths
- [ ] The protocol is still stated in exactly one place
