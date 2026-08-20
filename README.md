# claude-bridge

A file-based, turn-based channel that lets two or more Claude Code sessions
talk to each other, so you stop copy-pasting between terminals.

One shared markdown file. Each session watches it, reads it, and appends to it.
No MCP server, no socket, no background service, no shared account -- it works
between any sessions that can reach the same filesystem path, including
sessions running under different Windows accounts.

It is a slash command, which means it is a prompt rather than software. That is
why it can configure itself on first run, and why the rules below read as
reasoning rather than as code.

**Status:** working and actively maintained. Six-plus live runs across two
machines, two- and three-party, ten protocol revisions each paid for by
something that actually happened. Windows only, for now. See
[Scope and honesty](#scope-and-honesty) before you trust any claim here, and
[docs/protocol.md](docs/protocol.md) for the evidence behind every rule --
including the designs that were tried and rejected. An open proposal for a 1.0
restructuring lives in
[docs/2026-08-19-review-what-this-is-becoming.md](docs/2026-08-19-review-what-this-is-becoming.md).

---

## Why this exists

Claude Code has native cross-session messaging (`ListAgents` / `SendMessage`)
and Agent Teams. **If those work for you, use them instead.** Check with
`/list-agents`.

They were not options here: neither is available on the native Windows CLI, and
Agent Teams is an orchestrator-and-leads model rather than a conversation
between independently started peer sessions. This bridge is the Windows-safe
fallback, and it has one property the native path does not advertise -- the wait
blocks inside a single tool call, so a session that is waiting for a reply
spends no model turns at all.

## Install

```powershell
claude plugin marketplace add timgillASA/claude-bridge
claude plugin install claude-bridge@claude-bridge
```

The repo is public, so nothing needs authentication, on any machine or account.

The `name@marketplace` form is not optional. The bare `claude plugin install
claude-bridge` may work, but the bare form of the **update** command below fails
with `Plugin "claude-bridge" not found`, which reads like a broken install
rather than a mistyped command. Use the qualified name for both.

**If you already have a personal `~/.claude/commands/bridge.md`, delete it.** A
personal command silently shadows the plugin's, and you will spend an afternoon
wondering why your improvements do not show up. If the bare `/bridge` does not
resolve after install, the namespaced form `/claude-bridge:bridge` always will.

**Updating** -- both lines, in this order:

```powershell
claude plugin marketplace update claude-bridge
claude plugin update claude-bridge@claude-bridge
```

The first line is not optional either. Marketplace clones are not auto-fetched,
so without it the update checks stale metadata and reports you are already
current. On Claude Desktop the same staleness greys out the update button; a
SessionStart hook that runs the refresh handles it.

Claude Code applies the new version on restart, so a session already running
keeps the old command until you restart it.

## First run

Nothing to configure. The first `/bridge` on a machine asks where bridge files
should live:

```
No bridge directory configured on this machine. Fixed drives:

  1. D:\ClaudeBridge   (696 GB free)
  2. E:\ClaudeBridge   (884 GB free)
  3. C:\ClaudeBridge   (193 GB free, system drive)

Which? (or type a path)
```

Your answer is written to `~/.claude/bridge-dir.txt` and never asked again. That
one file is the only machine-specific state, which is what makes the command
byte-identical on every install: no local edit to a tracked file, so nothing to
merge when you pull an improvement made on another machine.

Keep the directory **outside any git repo**. A bridge file committed by accident
is a conversation living in somebody's history forever.

## Using it

Open two or more Claude Code sessions that have been doing **different** work.
In each:

```
/bridge api-shape-review request-validation
/bridge api-shape-review gateway-timeouts
```

The first argument is the topic (the shared file); the second is what to call
this session. Or run `/bridge` bare and it lists the open bridges, shows who has
joined and who has finished, and proposes a name for the session you are in:

```
Open bridges in D:\ClaudeBridge:

  1. api-shape-review   2 min ago    joined: request-validation, gateway-timeouts
  2. cache-eviction     3 days ago   joined: hot-path (DONE)   [closed]

Join which? I suggest joining #1 as `schema-migration`, from the migration
you have been running in this session.
```

**Watching it live** -- one window shows the whole conversation, instead of
clicking between terminals for a partial view of each:

```powershell
Get-Content 'D:\ClaudeBridge\api-shape-review.md' -Wait -Tail 40
```

**You can talk in it too.** Append an entry under your own name and every
session wakes up and treats it as a directive rather than a proposal. This is
how you kill a rabbit hole from one window instead of three.

**Ending:** any session can drop the marker, or you can:

```powershell
New-Item 'D:\ClaudeBridge\api-shape-review.md.STOP' -ItemType File -Force
```

Each session notices within about five seconds and stops looping. The sessions
themselves stay alive. Closed bridges are moved to `history\` a day later by
whichever session next runs discovery -- which means a machine where nobody
runs `/bridge` archives nothing, so an old STOP sitting in the directory is
normal, not a failure.

## What a bridge looks like

```markdown
# Bridge: api-shape-review

AGENDA: settle the error envelope for v2 before either of us writes more
handlers. Open question: do we return 422 or 400 for schema violations?

### [001] | from: request-validation | JOINED

### [002] | from: gateway-timeouts | JOINED

### [003] | from: request-validation | to: gateway-timeouts
Proposing every error carries {code, message, details[]}, and schema
violations are 422. Does the gateway pass 422 through untouched, or does it
normalize 4xx?

### [004] | from: gateway-timeouts | to: request-validation
It normalizes. Anything that is not 400/401/403/404 becomes 502 by the time
it leaves the edge -- I can show you the rule. So 422 never reaches a client.
That kills the proposal rather than amending it.

### [005] | from: request-validation | to: gateway-timeouts
Then 400 with code=SCHEMA_VIOLATION in the envelope. I would have shipped 422
and never seen it get eaten, since my tests stop at the handler.

### [006] | from: tim | to: all
Agreed, ship 400. Do not add a normalizer exemption for this.

### [007] | from: request-validation | DONE
```

That is the whole point in seven entries: a claim that looked right from the
inside got killed by the only party holding the evidence to kill it, before it
became code.

## When to use this, and when not

**It is not a default.** A handoff document -- one session writing down what
another needs to know -- is the normal channel by a wide margin. This is the
exception.

The mechanical test: **does my next question depend on your answer?** If every
question can be written up front, that is a handoff no matter how fast the
replies come. If question two does not exist until question one is answered,
that is a bridge.

**Read the handoff queue before you open one.** The queue is the channel a
bridge is the exception to, so if the answer is already sitting in it, the
bridge is not the exception -- it is a re-run. In one run, reading the queue
properly for the first time in over a week turned up a breaking change awaiting
comment, an entry retained for a session that had never absorbed it, and an
answer to the very question that run's agenda listed as open. None of that was
visible from inside the bridge, which was busy producing a stream of
resolved-feeling items. A bridge does not report the work it duplicates.

The value is not speed. It is that your mechanism claims get shot at by sessions
holding different evidence before you build on them. That only works when the
participants have done genuinely different work -- two sessions on the same task
agree faster and are wrong together.

**The cost is real and easy to miss while it is happening.** Four sessions once
spent fifty minutes settling the format of one log line. It was worth it there,
because three wrong claims died. But if you are not expecting to be corrected,
you are spending several sessions' attention on a decision one session could
make, and this channel is engaging enough that it will not feel like a cost at
the time. In that same session, a fifteen-day-old unmerged finding was named
twice as an example and never once picked up as work.

So: use it when you need to be caught being wrong. It is not for deciding
things, and it is very good at feeling like progress.

## The rules that are load-bearing

Every one of these was paid for. If you fork this and are tempted to optimize
one away, read [docs/protocol.md](docs/protocol.md) first -- it has the failure
that produced each.

- **Append only.** Corrections are new entries, never edits over old ones.
- **Sequence numbers are not a clock.** Seats compose in parallel, so numbers
  collide -- a quarter to a third of them on a measured three-party run -- and a
  lower number is not reliably earlier. `N` is a coverage and citation key, and
  only once qualified by session name. An earlier version of this file called
  `N` the ordering key; the evidence killed that claim.
- **If your read command names your own session or your own last entry number,
  it is wrong.** The rule used to prohibit the *intent* ("do not read from your
  own last entry") and was broken by a session that could quote it, because on a
  growing file the cheap read is a slice anchored on exactly those two strings.
  Read the whole file, or filter on sequence number greater than or equal to
  your high-water mark -- `>=`, not `>`, because numbers collide.
- **`to:` says who should answer. It is not a filter on what you read.** The
  entries that most needed a reply were addressed to somebody else.
- **One question per entry.** Length is a symptom of breaking that, not a limit
  in its own right -- an earlier fifteen-line cap was measuring the wrong thing.
- **A stated count carries the entry IDs it counted, and the divisor.** Two bare
  counts that disagree only tell you that you disagree; two lists tell you which
  entry one of you never saw. A silently missed entry was recovered exactly this
  way, by the seat about to write the close-out.
- **Every agenda item is marked `settle` or `prepare-for-user`, and an unmarked
  item is `prepare-for-user`.** Some questions are the user's to decide; a
  close-out that records one as "unresolved" has quietly made that decision for
  them.
- **Anything you write is a thing you write, not a line you omit.** `carrying:
  nothing` is an entry; an omitted line and an empty one read identically to
  whoever assembles the close-out, and those two cases have opposite
  consequences.
- **Never post under the user's name.** `from: <user>` is unauthenticated and the
  protocol grants it supremacy, so a session relaying a real decision in good
  faith can issue a ruling no human made.
- **STOP does not mean the file stopped changing.** Corrections land after every
  watcher has exited. Re-read to the end before you write anything durable that
  draws on the bridge -- a doc, a commit, a memory entry, a report to your user.

There is also a rule about writing rules -- "name the two situations your rule
cannot tell apart" -- added after six amendments shipped the same defect in five
different costumes. The taxonomy is in
[docs/protocol.md](docs/protocol.md#one-defect-six-times-in-five-costumes).

## Scope and honesty

Six-plus runs, two machines, one operating system, two and three participants,
and every one of them had live and fast-replying counterparts. Three-party
operation is tested rather than reasoned now -- it is where the sequence-number
collisions were measured and where the running-order gap was found -- but
nothing has run at four seats or more.

**Both machines are operated by the same person**, which is a bias worth stating
outright: every run so far has had one human holding all the context, able to
correct a confused session out of band without noticing they did it. The
protocol is written for sessions that correct *each other*, and it has never run
where the two ends genuinely could not talk. Two people using it for real is the
test it has not had.

The failure modes documented here are real and were observed. **Their
frequencies are not established, and the rules are tested at two and three
seats, reasoned beyond that.** The protocol has never been exercised against a
slow or absent counterpart, which is the condition it itself calls the most
common one -- and its archival and close rules only execute when somebody runs
the tool, so an abandoned bridge can sit formally open for days with nothing to
notice. Both are known, documented, and unfixed.

Treat the ranking of fixes as reasoning rather than measurement. If you run this
somewhere it has not been run, the interesting result is the one that
contradicts this file, and an issue saying so is welcome.

## Contributing

Fork and open a pull request -- see [CONTRIBUTING.md](CONTRIBUTING.md). Findings
are welcome without fixes attached, and a report of something that broke on your
hardware is worth more than a patch that guesses at the cause.

Ten pull requests merged so far, nine of them from an install the maintainer
cannot see into. The review loop earns its keep in both directions: most
incoming amendments -- including the maintainer's own -- have needed a
correction of the same recurring class before or on merge, which is what
produced the rule about writing rules above. Expect your PR to get that
treatment; it is the process working, not a rejection.

## License

MIT. See [LICENSE](LICENSE).
