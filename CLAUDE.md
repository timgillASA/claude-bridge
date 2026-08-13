# claude-bridge -- working rules

This repo is **public**. Everything committed here is permanent and readable by
anyone.

## Before you commit

- **No internal identifiers, ever.** No employer or org names, no internal repo
  or server names, no ticket IDs, no paths that reveal an internal layout, no
  specifics of an internal security finding. The evidence in `docs/protocol.md`
  is deliberately anonymized -- keep new evidence that way. "Four sessions spent
  fifty minutes on one log line" is publishable; naming the four is not.
- **ASCII only.** No em-dashes, en-dashes, smart quotes, or emoji. Use `--` and
  `->`. These files get round-tripped through PowerShell and shells that mangle
  the rest.

## Structure

- `commands/bridge.md` -- **the only copy of the protocol.** Never restate the
  rules anywhere else. An earlier version of the docs embedded a second copy of
  the command and the two drifted within a day.
- `docs/protocol.md` -- why the rules are the way they are, plus run history and
  rejected designs. Explains the command; never duplicates it.
- `README.md` -- the front door: what it is, install, examples, honest scope.

## Changing the command

1. Edit `commands/bridge.md`.
2. If the change came from something that actually happened, add the evidence to
   `docs/protocol.md`. A rule with no failure behind it is a rule the next
   person will optimize away.
3. Bump the version in **both** `.claude-plugin/plugin.json` and
   `.claude-plugin/marketplace.json`. They must match.
4. Commit and push. Other machines pick it up with `claude plugin update
   claude-bridge`.

## Two rules that keep getting rediscovered

- **A personal `~/.claude/commands/bridge.md` silently shadows this plugin's.**
  If an improvement does not appear after an update, look for a local copy
  first.
- **`bridge-dir.txt` is per-machine and untracked.** Nothing in this repo may
  hardcode a bridge directory path, or the command stops being byte-identical
  across installs and every machine grows a permanent local diff.
