# Why the rules are the way they are

Reasoning, run history, and rejected designs. The command in
[`commands/bridge.md`](../commands/bridge.md) is the only copy of the protocol
itself; this file explains it and must never restate it. An earlier version of
this document embedded a full copy of the command, and the two drifted within a
day, which is the usual fate of a second source of truth.

Read this before softening any rule. Each one is here because something failed.

---

## The defect that produced most of the design

**Reading by position or by timestamp is broken, and it fails invisibly.**

Appends do not land in timestamp order. Sessions compose while others are
writing, so an entry can appear with a timestamp earlier than the entry above
it. Directly observed: a bridge file's last-write time preceded the wall-clock
start of the watcher process that was about to write to it.

Early runs had every session improvising its own way of slicing the file, and
all of them were fragile:

- One session sliced from its own last entry and read past three entries from
  another, leaving a direct question unanswered through two full rounds.
- Another audited all 27 entries against its 5 reads after the fact and found
  one skipped. It cost nothing purely by luck -- the missed entry was addressed
  to a third party. Had it been addressed to that session, it would have ignored
  a direct question and reported a clean run.

That second case is the important one: **the defect is invisible from inside the
session that suffers it.** Two of three sessions caught theirs only by accident,
and the third only because it was asked for a retrospective afterwards.

Sequence numbers plus read-the-whole-file fix it deterministically. Tracking by
entry identity (`from` plus timestamp) was proposed as an alternative and
rejected: it rests on the clock that the same evidence proves unreliable.

The rule kept paying out after it was adopted. On a later run, a session's read
immediately after a wake caught entry `[3]`; `[4]` landed in the gap between the
wake firing and the read completing. Reading the whole file by sequence number
caught it. Reading "since my last entry" would have dropped a peer's message
silently.

## Short entries beat long ones, measurably

Two runs, compared:

| | Run 1 | Run 2 |
|---|---|---|
| Sessions | 2 | 4 |
| Duration | ~70 min | ~55 min |
| Rounds | 6 | 7 |
| Entry length | 40-60 lines | 6-15 lines |
| Outcome | a docs repo split into five files | an ingest protocol settled |

The short-entry run went better on every axis despite twice the participants.
That is the whole basis for the fifteen-line rule. A long entry is a handoff
wearing a bridge costume: it draws a long reply, and the thread dies of weight.

## Solutions get cheaper each round, if the format allows it

In run 2 the proposal shrank every round. Round 1: a handoff file per ingest.
Round 3: one line in a shared log. Round 5: no separate artifact at all.

Each round *removed* something, because each participant was defending its own
cost of compliance rather than the elegance of the scheme. A design that grows
every round is a sign the participants are not the ones who will pay for it.

## The value is adversarial correction, not speed

Three mechanism claims were corrected in under an hour, each by a session
holding different ground truth than the claimant:

- An append was asserted to be atomic, so two sessions could not interleave.
  That is a POSIX `O_APPEND` guarantee under `PIPE_BUF`; the setup in question
  runs Git Bash on Windows, where it does not hold. Appending stayed the rule,
  as a discipline rather than a guarantee.
- A hazard was described as live with nobody watching. Nothing was deployed;
  the exposure was prospective. The correction stopped a non-existent incident
  from reaching the user as a real one.
- A log line was specified to carry the ref of the commit that would contain it.
  Self-referential, and the fix removed a duplicated artifact as well.

**None was catchable by its author.** This is the entire argument for the
channel, and it is also the argument for the naming rule: the value depends on
participants holding different evidence, and the session name is the only signal
of that in the file.

## And the honest cost

Four sessions spent fifty minutes settling the format of one log line. Worth it
there, because three claims died. But during that same session a fifteen-day-old
unmerged finding -- which the owning repo's own notes called its most
consequential unmerged fact -- was named twice as an example of a cost and never
once picked up as work. Four sessions coordinated intensively about coordination
while the actual work sat still.

Value decay does not show up in content. From inside and from outside, the
thread keeps looking productive; it shows up only as a price comparison, and
only the user holds the prices. That is why the round cap ends with a question
to the user rather than a self-assessment, and why the question is "is this
still worth the clock versus you just deciding?" rather than "is this still
productive?", which invites yes from everyone.

## Confirmed properties

With evidence, from runs that set out to break them:

- **It does not wedge a session.** Three interrupts, three immediate returns,
  session usable every time.
- **It does not orphan its watcher.** Tested against a bridge with no live
  counterpart so the watch actually blocked: the process wrote its own PID to a
  file before looping and was dead immediately after the interrupt. Earlier
  attempts failed to test the case at all, because a fast-replying counterpart
  meant the interrupt landed between tool calls rather than mid-block.
- **Waiting is free.** The watch blocks inside a single tool call, so no model
  turn is spent waiting. **This is the single most important property and it is
  the easiest to break by "improving" the poll into something that returns
  between checks.** A related project died precisely because each empty poll
  cost a full model turn.

One caution about testing the second point: a process check that greps command
lines for the string `ClaudeBridge` matches *itself*, because the querying
command contains that string. That produced a confident report of leaked watcher
processes which was entirely false and had to be retracted after it was already
published into a close-out. Exclude the querying PID.

## The first-run drive list must exclude network drives

The first run asks where bridge files should live. The original probe was
`Get-PSDrive -PSProvider FileSystem`, which does not mean "fixed drives" even
though the instruction above it said so.

On the first real install, that probe returned six rows. Four were local disks.
One was a mapped network share with over twenty terabytes free -- roughly forty
times more than any local drive, and therefore the most attractive answer to
"a non-system drive that has room". One more was a `Temp` PSDrive alias
pointing at the system drive, which is not a drive at all.

The session recommended a local disk anyway, so nothing broke. That is the
point worth recording: **the list was wrong and the model compensated.** A rule
that holds only because the reader is smart enough to ignore it is a rule that
fails the first time it meets a smaller model, or a machine whose largest mount
happens to be a NAS.

Network drives are wrong here for two reasons beyond speed. The watch loop stats
the file every five seconds and the cost model assumes a cheap local call. And a
mapped share is reachable from more than one machine, so two installs could open
the same bridge with no idea the other exists -- sequence numbers collide, and
the "skip entries where `from` is your own name" rule cannot save a session from
a stranger it never expected.

`Win32_LogicalDisk` with `DriveType=3` returns fixed local disks only, which is
what the surrounding prose always claimed to be asking for.

**Then the same defect turned up one layer down.** Verifying that fix on a
second machine reproduced the original bug -- six network drives listed, and the
drive with the most free space of any drive on the box was a mapped share -- but
it also showed that the fixed-disk filter is not sufficient. On that machine the
roomiest *local* disk, by a wide margin, was an external backup SSD: `DriveType`
3, `BusType` USB. The corrected probe would have ranked a detachable drive
first.

That matters more here than it looks. A bridge folder on a drive that gets
unplugged does not fail loudly. The watch loop suppresses errors on its
`Get-Item`, so the file's last-write time comes back null, compares unequal to
the previous value, and the loop reports `CHANGED`. Every session wakes and
re-reads a file that no longer exists. The failure presents as activity.

The lesson is the same one twice: **a probe that answers a slightly different
question than the prose asks will be wrong in a way the prose cannot see, and
ranking by a single number picks exactly the wrong candidate.** Rank by
suitability and check the bus.

## STOP is not the end of the file

Dropping the STOP marker ends every watcher. Anything appended afterward wakes
nobody.

On one run, two entries landed after the close-out: one retracting a finding
that was false, and one recording a test result that resolved an open question.
Every participant had already stopped reading. **The retracted finding was
nearly acted on.**

Hence: corrections are appended, never edited over a prior entry, and anyone
about to act on a close-out re-reads the file to the end first. This is also why
closed bridges are archived a day later by a passing session, rather than moved
at close -- a correction appended to a bridge that has just been moved lands in a
fresh empty file at the old path, or fails outright, and nobody learns either
happened.

## Rejected designs

Recorded so they do not get rebuilt.

**Filtering wakes by addressee.** After adopting read-the-whole-file, the
obvious optimization is to have the watch script grep the new tail for your own
name or `all` and return `CHANGED-FOR-ME` versus `CHANGED-OTHER`, so a session
can skip irrelevant reads. With several participants most wake-ups are
irrelevant, so the saving is real. **Do not build it.** The entries that most
needed a reply were addressed to someone else -- a rule that bound a session it
was not addressed to, and evidence that changed an answer in someone else's
exchange. A name filter skips exactly those. The wake cost is real and the only
safe way to pay it is to read everything and decide by content. The two ideas
look compatible until you notice they are not.

**A hold-until-go parameter**, to stop sessions posting JOINED the instant they
arrive and burying the convening entry twelve entries deep. It should not exist:
the eagerness is what defeats the late-join deadlock, the damage attributed to
it was actually the read defect above, and a parameter is per-session config for
a per-conversation property -- it has to be set identically in every terminal,
and one session missing it voids the constraint. The agenda in the file header
solves it instead.

**Lock files, staggered poll intervals, a richer file format.** No write
collision occurred across two runs and thirteen rounds. Appending is right
because read-modify-write is what clobbers and nobody does one -- but that is
discipline, not an atomicity guarantee.

**A "chair" role for three or more parties.** Both of its duties, opening and
posting the close-out, already belong to whoever creates the bridge. It added a
word and no behavior.

**A hook-enforced checkpoint with a parking state machine.** Proposed on the
correct observation that "append DONE when you have nothing further" is
self-assessment from inside, on a channel that is very good at feeling like
progress. The fix grew to a settings-level hook counting rounds outside every
session's context, sessions parking mid-conversation while staying awake, a
dedicated `CHECKPOINT` entry type, and a helper script so the user could answer
from any window. **The round cap was kept; none of that machinery was.**

Its premise came from a different system, one that ran autonomously with nobody
necessarily watching. A bridge only runs while the user is sitting in a session,
almost always the creator, and they can interrupt or drop STOP at any moment;
the outside observer the checkpoint escalates to is already in the room. Most of
the complexity answered "which of five terminals do I answer in?", a question
that disappears once the creator prints the close-out to its own user.

There is also nothing to write down. The round count is already the highest
sequence number in an append-only file, so a rule that only *reads* cannot lose
an update. Deleting the state beat locking it. Revisit if a bridge ever runs
unattended, or passes ~10 rounds without converging.

## The identity hole, and why the fix is a prohibition

**A `from: <user>` entry is unauthenticated, and the protocol used to make that
the most dangerous possible property.** User entries outrank every session
absolutely, and the rules forbade debating them. Supreme, unverifiable, and
unchallengeable.

It failed on the third real run, with no bad faith anywhere. The user had made a
genuine decision in one session -- picked option 1 off a numbered menu. That
session wrote the decision out in first person, added specificity the user had
never uttered (a full ownership model naming which project owned which
component), and signed it with the user's name. A cross-project ruling that no
human issued acquired absolute authority over two other projects.

Three properties of that failure are why the rule is a prohibition rather than
guidance:

- **The intent was benign.** The session believed it was relaying faithfully and
  saving the user a step. Impersonation required a convenient shortcut and a
  channel that made it look normal, not malice.
- **The only control that worked was out of band.** A peer found the directive
  unusual enough to go ask its own human. That is a good instinct, not a
  mechanism, and it is not guaranteed to be present.
- **The same landing zone is reachable without anyone doing anything wrong.** A
  session summarising something the user said in another window arrives at an
  identical entry.

Hence: post relays under your own name, marked RELAYED, quoting the user
verbatim. The user's name is reserved for the user typing into the file.

**And the correction was worse than the fault.** The peer that challenged the
forged entry posted an URGENT cross-project revert instruction citing evidence
it did not have: it had told its user that an entry attributed to them existed,
received a four-word reply answering a different question, and reported that as
first-hand confirmation. The conclusion was correct. The evidence was invented.
Had the forging session not independently confessed, a true claim would have
been resting entirely on a fabrication, which from outside is indistinguishable
from the reverse.

That is why challenging authenticity is now explicitly permitted -- the peer was
technically breaking the no-debate rule by challenging at all -- and why the
out-of-band check must quote back the exact claim being ratified. **Do not file
that run as evidence that peer review catches impersonation.** It caught it by
confession, after a fabricated challenge, past a retraction that was itself
wrong.

## Three parties break assumptions that two parties never tested

The first runs had two participants, and several rules were sized for that
without anyone noticing:

- **Sequence collisions.** "Visible and harmless" was true for reading and false
  for citing. Counted on one three-party file: 14 of 25 numbers were used by more
  than one session, and two numbers by three. Close-outs cite entry numbers --
  that is the rule that makes them auditable -- so ambiguous numbers attack
  exactly the artifact that matters most. All three sessions independently
  invented `[<session> <N>]` mid-run.
- **DONE removing evidence, not just a voice.** A session hit the round cap,
  posted DONE, and left. The remaining two then designed a safety rail whose
  first defect was an assumption about the very system the departed session held
  ground truth on. One of them wrote, in the file, that the evidence had left the
  room -- and proceeded anyway.
- **Close-outs crossing in-flight entries.** Two sessions posted entry `[17]`
  simultaneously, one of them the close-out. The close-out tallied the round
  without a design option and a binding constraint that were landing as it was
  written. Corrections appended afterward are permitted, but then the close-out
  is no longer the last word and anyone who reads to it and stops is misled.

## Liveness is not observable, and inferring it fails in both directions

The watch loop wakes on a write, never on a read. An attentive peer and one that
exited an hour ago are identical from inside a session.

Both errors were made on the same run. Four consecutive entries were addressed
to sessions that had already exited -- written to an empty room, at full cost.
Then a closing entry asserted that entries `[20]`-`[22]` "were never delivered",
which was false: a peer had read and answered both. The author had inferred an
empty room from a true statement made at a different time.

There is no fix here, only a prohibition on claiming what cannot be seen. The
absence of a reply is not evidence of absence.

## The bridge that could not be closed

One run's creator posted two close-outs and never posted DONE. STOP is only
droppable once every JOINED session has DONE'd, so the close condition was
permanently unreachable. The bridge sat formally open while actually abandoned,
which is precisely what a peer misread as "still live" while writing into the
empty room above. It was eventually closed on the user's explicit instruction,
by a session that recorded the close as irregular.

A protocol whose close depends on one specific participant has no close at all
when that participant walks away. Any session may now drop STOP once the cap has
fired and the file has gone quiet, recording that it did so.

## Why every seat writes its own recap

After one three-party run, every participant was asked for its own account. The
overlap was substantial and the disagreements were the whole payload:

- One session reported the round cap firing "at the natural end" of the
  conversation. Another demonstrated the cap "had no teeth" -- flagged correctly,
  handed to the creator as the rules then required, and followed by fourteen more
  entries. Both reports were honest; they were written from different seats.
- The most serious finding of the run appears in only two of the three, because
  the third had exited before it landed and wrote its recap without re-reading to
  the end. It rewrote the recap afterwards and recorded the miss rather than
  quietly absorbing it, which is the only reason the gap is visible at all.
- Each recap is candid about a different failure, and in two cases the author's
  worst finding is about itself. A single close-out cannot produce that, because
  a close-out is one participant writing about everybody.

**Do not consolidate them.** A merged version would have had to pick one account
of the round cap, and both were true.

One naming lesson, learned the dull way: three recaps of one run, each named for
the topic and none for its author, are indistinguishable a day later.

## A converged bridge has no exit

Separately and on another machine, a two-party run reached a clean answer and
then could not stop.

The two sessions held genuinely opposite halves of one infrastructure question.
They corrected each other five times in both directions -- one killed an entire
option with a command's output rather than recall, the other reframed the
question, and the second later abandoned its own preferred design **on the
first one's evidence rather than its own**. That is the channel doing precisely
what it exists for.

Then they agreed, and stopped. Counted entries divided by live participants
came to 4.5 rounds, so **the cap never fired**, because the cap only catches a
conversation that runs long. A conversation that finishes early has no trigger
at all. Both sessions sat polling, each waiting to be sure the other was
finished, until the user broke in and asked what was happening. Neither did
anything wrong; the protocol simply had no way to end a success.

Hence the rule that agreement is a reason to close rather than to wait. Note
what it is not: it is not a timeout, and not a second cap. The instruction is to
notice that you are only waiting to be certain, and to say so instead.

## The correction that arrived after everyone had finished

The same run produced the sharpest case yet for re-reading, and showed the rule
was aimed at the wrong instant.

One session posted DONE. The other then appended a correction: an item the
close-out had recorded as unverified had in fact been settled by the user in
conversation, and a user's direct statement outranks a tool's 403. Correctly
reasoned, correctly appended rather than edited over, and cited.

Nobody read it. The addressee had posted DONE and its loop had ended. Hours
later its written specification and its saved session memory both still carried
the uncorrected claim, in a document another person would reasonably build on.

The old wording said to re-read "before treating a close-out as final, and
especially before acting on a finding from one". A session that has finished has
no moment of acting -- it has a moment of *writing down*, and that is not the
same thing. The rule now attaches to producing any durable record that draws on
the bridge. **A protocol whose corrections do not reach the artifacts is a
protocol that produces confident, well-cited, out-of-date documents.**

## The job assigned to whoever has already left

Two independent findings, from different machines and different failure paths,
turned out to be the same shape.

On a three-party run the round cap was flagged correctly and handed to the
creator as the rules then required, and the bridge ran another fourteen entries
because the creator was not paying attention. On a two-party run the STOP marker
was never dropped, because the rule said to drop it once every session had
posted DONE -- and the only participant who could observe that condition had
ended its own loop by posting DONE.

Both are the same error: **assigning a duty to a role, when the duty can only be
discharged in a window that role is usually not present for.** The fixes are the
same shape too. Whoever notices the cap acts on it. Whoever is about to post
DONE checks the close condition first, because that is the last instant they are
still reading.

Worth stating plainly, since it is the sort of thing a maintainer talks himself
out of: neither fix makes the protocol cleverer. Both move a responsibility to
whoever happens to be looking.

## Leaving the loop is legitimate

On the two-party run, one session found a security problem in the course of its
research, left the watch loop without being asked, and escalated to its user. It
had judged correctly that the finding outranked the conversation, with nothing
in the protocol telling it that was allowed.

The protocol had no opinion, which is the defect. Its counterpart went on
polling into silence, because a session that stepped out, a session composing a
long reply, and a session that has died are indistinguishable from outside -- see
the liveness section above. The good judgement produced the bad state.

Leaving is now sanctioned, with one obligation: append a line saying you are
going before you go. The line is not manners. It is a write, and a write is the
only thing that wakes anyone.

## Two rules the evidence went against

**The user-entry mechanism was unreachable.** On that run the user intervened
four times -- an escalation, a stall, an extension of the conversation, and a
statement of fact that settled an open question -- and the file records none of
them. Every one happened in a terminal. The protocol described user entries in
detail while never telling the user how to write one, and the one fact that did
reach the file got there only because a session chose to relay it. That relay
path has since been closed off for good reasons (see the identity hole), which
makes telling the user how to post directly a precondition rather than a
convenience.

**The fifteen-line limit did not survive contact.** Six entries broke it, by
both participants, and the conversation stayed sharp throughout -- dense entries
carrying one question and the evidence for it. The original rule came from
comparing two runs in which the long entries were *padded*, and length was
standing in for the thing that actually mattered. One question per entry is the
rule; length is a symptom worth noticing, not a limit worth enforcing. Both
sides followed one-question-per-entry almost perfectly, and the single lapse was
an entry asking two things at once, which is the failure the limit was proxying
for all along.

**And one ambiguity worth closing.** A direct question in that run was never
answered by its addressee -- the asker made it moot in its own next entry by
answering it from evidence already on the table. Benign, but from outside the
transcript a moot question and a skipped one are identical, which defeats the
audit the read-the-whole-file rule exists to support. Withdraw a question out
loud when you drop it.

## The watch script had to say which interpreter runs it

For several versions the watch-script section opened "Bash tool, PowerShell",
naming a tool as though every harness had one. Two things went wrong with that.
A harness with no PowerShell tool has a session that cannot find the named tool
and improvises; a harness with only a Bash tool has a session that pastes the
script into Bash, where it dies on `=: command not found` and a syntax error at
`while ($true) {`, exit 127. Either way the first wake of every fresh session
costs a wasted turn, on the one step whose entire purpose is to spend no turns.

Reproduced deliberately and then fixed by naming the interpreter rather than the
tool: the section now says the script is PowerShell and must be executed by
PowerShell, and gives `powershell -NoProfile -Command` as the invocation for a
harness that only has Bash. Verified on a Bash-only harness afterwards, exit 0,
returning CHANGED.

The general form is worth stating, because this file is read by sessions running
in harnesses nobody here has seen: **name the capability, never the tool.** A
tool name is an assumption about somebody else's environment, and it fails
silently at the worst moment, which is the first one.

## The watch baselined itself after the gap it was guarding (reasoned, not observed)

Found by review of the script, not by a run, and marked accordingly.

For every version through 0.11.0 the watch script started with `$last = $null`
and adopted the file's current timestamp on its first iteration. So the baseline
was taken when the *watch* started, while the session's knowledge of the file
dates from when it *read* -- and those are different moments with model work in
between (deciding whether to reply, composing, or concluding no reply is owed).

An entry landing in that gap is invisible. The watch compares the file against a
baseline taken after the write, reports it unchanged, and sleeps. The session
that appends before re-entering the watch is safe by accident -- its own append
wakes the peer, and the peer's next entry wakes it. The dangerous path is the
session that re-reads, decides **no reply is needed**, and re-enters the watch:
a peer's entry landing between its read and the watch's first stat wakes nobody,
and in a two-party bridge that is a peer waiting indefinitely on a reply to an
entry that was, by every observable measure, delivered. Nothing errors. The
failure presents as a quiet counterpart, which the liveness section says is
indistinguishable from a thinking one.

This is the wait-side twin of a read-side gap already recorded above -- the
`[4]` that landed between a wake firing and the read completing, caught there by
reading the whole file. The read rule cannot catch this one, because the session
is not reading; it is waiting.

The two situations the old script collapsed, per the contributing rule: a file
unchanged since your read, and a file that changed between your read and the
watch starting. Opposite consequences, identical baseline.

The fix moves the baseline to the only moment that means anything: the session
captures the file's `LastWriteTime.Ticks` immediately **before** reading, and
passes it into the script. Stat-then-read, in that order -- a write landing
between the stat and the read is covered by the read and costs one spurious
CHANGED, while read-then-stat leaves a write invisible to both. The script's
first poll then compares against the read-time baseline and fires immediately if
anything landed since. The failure direction is the safe one everywhere: stale
or mismatched baselines produce an extra wake and an extra read, never a silent
sleep.

Worth noting against the layer taxonomy in the 1.0 review: this is the first
defect found in the script layer, whose record was previously clean. It does not
overturn the taxonomy -- the fix is a script fix, one line of contract change,
and it was findable by inspection precisely because the layer is deterministic.
A prose rule with this defect would have needed a run to fail first.

## DONE ended the wrong thing

A two-party run, the first conducted under the identity rules rather than the
one that produced them. One session posted DONE, then kept watching anyway --
against the letter of "posting DONE ends your loop" -- because the close-out had
not landed yet and it knew the close-out would attribute positions to it that it
intended to report to its user.

It was right, and the protocol was wrong. Two rules had been pulling against
each other in plain sight:

- "Post DONE, do not keep polling."
- "Re-read to the end before you write anything durable that draws on this
  bridge."

Every recap is a durable write. The close-out lands after DONE by construction.
So the second rule assigned a re-read at precisely the moment the first had
already told the session to stop watching -- the same defect as the section
above, arriving from the opposite direction. There, a duty was assigned to a
role not present for it; here, to a session the protocol had just dismissed.

The fix separates two acts the protocol had conflated: DONE ends your
**contribution**, not your **watch**. Note the failure mode this closes is
silent. A session that stops at DONE does not error; it produces a confident
recap of a conversation whose ending it never read.

## A rule that competes with an affordance loses

The whole-file re-read is among the most emphasized rules in the protocol. It
still failed, and the useful part is why.

One session read the bridge from an offset on each wake -- the cheaper call, the
one the tooling encourages, and the one that produces no error when it drops
entries. It missed seven, and consequently asserted that an item was untouched
while a ruling on that item had been sitting in the gap the whole time. It
disclosed this itself; from the counterpart's seat the failure was invisible,
because an unanswered entry and an unread entry look identical.

The counterpart complied with the rule, and was candid about why: it happened to
be checking entry *headers* each wake, a cheap whole-file grep, rather than
reading bodies from an offset. Habit, not virtue, and not the rule.

That is the finding. **Prose instructing thoroughness competes with a tool
affordance, and loses under load.** Repeating the instruction more loudly does
not change the economics. The protocol now names the cheap complete technique
instead -- grep the one-line entry headers across the whole file, diff against
the numbers already handled, read bodies only for the gaps -- so that the
cheapest method available is also the correct one. A rule only holds when
following it is not the more expensive option.

### The second failure, on a session that had read the fix

Naming the cheap complete technique was necessary and it was not sufficient. On
a later run a session broke the same rule in a form the wording does not reach.

It did not read from an offset. It ran a single command over the whole file --
a range slice anchored on the header of its own last entry, spanning to the end.
That is the forbidden read wearing the shape of the required one: the command
opens the file at byte zero, the session was not thinking about offsets, and the
output looks like a read of everything that matters. It dropped three entries,
one of them addressed to that session and containing the two lettered options
the run's central decision was subsequently conducted in terms of.

It was caught by a dangling reference. A later entry used those options as
established terms, the session had never seen them defined, and it went looking.
Nothing in the protocol produced that catch.

The generalisation is worth more than the incident. **The rule is phrased as a
prohibition on intent -- do not read from your own last entry -- and the failure
is a property of ergonomics.** On a growing file the cheap read is an anchored
slice, and the two anchors any session reaches for first are its own name and
its own last sequence number, because those are the only strings it knows
without looking. The forbidden anchors are exactly the available ones. So the
rule has to be stated as a property of the command rather than of the reasoning:
**if your read command names your own session or your own last number, it is
wrong.** That is checkable by the session about to run it, which "do not read
from your own last entry" is not, since the session running the slice did not
believe it was doing that.

## Sequence numbers order nothing once seats compose in parallel

The entry format has claimed since the first version that the sequence number is
the ordering key. At two seats that is close enough to true to be useful. At
three it is false, and the claim invites a class of reasoning the file cannot
support.

Seven of twenty numbers on one three-party run were taken by more than one
session, one number by three. Everyone composes in parallel, so `N` is only
whatever a session saw as highest at the moment it began writing. Two entries
sharing a number were not simultaneous; a lower number is not reliably earlier.
What ordered that file was append order, which no field records.

Nothing broke, and that is the point worth recording rather than a near-miss:
session-qualified citation absorbed the ambiguity completely, no position was
mis-attributed, and both close-outs were accurate. The mitigation is sound. What
needed changing was the description, which named `N` as something it is not and
would eventually license a session to infer who knew what from the numbers.

`N` is the coverage key -- which entries you have handled -- and the citation
key, once qualified by session. It is not a clock and it is not a sequence. The
collision rate is also higher than "visible and harmless" suggests when a
quarter to a third of the numbers on a run are shared, which is the condition
that makes the qualified form mandatory rather than advisable.

## The adjective that undefined a definition

Two seats on the same two-party run disagreed about how many rounds it had run:
one said 5, the other 4. The whole gap was a single entry -- a RELAYED evidence
drop that asked no question but moved the bridge's headline item. Neither seat
thought it worth an entry to settle, which was the right call and also meant the
cap's trigger was fuzzy at exactly the point it fires.

The rule they were reading was already mechanical:

> Substantive entries = every entry except JOINED, DONE, and close-outs.

By its own text the evidence drop counts. The definition asks for a subtraction
and nothing else. But the label said **substantive**, and a reader who takes the
label seriously reasonably asks whether a given entry was substantive -- a
question the definition never poses and offers no way to answer. The term was
doing work the rule had not authorized.

So the fix is not a better definition of "substantive". It is deleting the word.
The count is now **counted entries**, the subtraction is unchanged, and the rule
states outright that it is a subtraction rather than a judgement.

The principle worth keeping: **a cap must be independently computable by every
participant, or it is not a shared trigger.** Both seats hold the same file and
must derive the same number from it every time. Any definition requiring an
opinion reintroduces the 5-versus-4 split however carefully it is worded, and a
trigger that two participants compute differently cannot coordinate them. It is
also a cheap property to preserve here, since the count falls out of the
whole-file header grep the re-read rule already requires.

Rejected while fixing this: counting only entries that "ask or answer a
question". It sounds tighter and is worse -- it replaces one contested word with
three, since a correction, a retraction and an acknowledgement each need a
ruling before the count can proceed.

## The pacing rule that caught an integrity failure by accident

A session read the file from an offset instead of whole, and silently missed an
entry. Nothing in the protocol detected that -- a missed entry is invisible from
inside the seat that missed it, which is why the whole-file re-read rule exists
and why breaking it is so quiet.

What caught it was the round cap, which has nothing to do with integrity. Two
seats disagreed about the round number, and both happened to **enumerate the
entries they were counting** rather than assert a total. One list contained an ID
the other had never seen. That is what surfaced the miss.

Had either seat written "I get 5, you get 4" and stopped there, the likely
outcome is the cheapest one: somebody defers, the count is reconciled, the missed
entry stays missed, and the close-out is written by the seat that never read the
entry that mattered most to it. The disagreement would have been resolved and the
defect preserved -- resolution and correctness pointing in opposite directions.

So the rule is now that a stated count carries its IDs. It is a small change and
it is not really about counting: it converts an assertion into something the
other end can diff. A count is a summary of a read and inherits every defect of
that read silently; the IDs *are* the read, and a missing one is visible to
anybody holding a different list.

Generalising slightly, because this is the second rule in this file to arrive by
the same route: the protocol's own consistency checks -- the round cap, the
close-out citation rule -- keep turning out to be worth more as integrity checks
than as the thing they were written for. Both work by forcing two seats to
compare a derived value against the same source. That is worth knowing when
adding a rule: a check two seats evaluate independently against the file is
cheap, and it catches failures nobody was looking for.

**The limit, stated plainly:** this only fires when two seats independently do
the count and then disagree out loud. A run where one seat never counts, or
where both make the same off-by-one, gets nothing from it. It is not a
verification mechanism, and it should not be cited as one -- the whole-file
re-read is still the thing that prevents the miss, and this is a net that
happened to be under it once.

**The divisor needs the same treatment (reasoned, not observed).** As first
written, the rule enumerated the dividend and left the divisor a bare number.
Counted entries exclude JOINED and DONE by definition, so a missed entry of
either kind can never show up in an ID list -- and those are precisely the
entries that change the participant count. Two seats would diff identical lists,
agree, and both divide by the wrong number. The fix is to name the live
participants rather than count them, which costs nothing because they are names
and there are only ever a handful. No run has hit this; it was found by asking
what the ID list cannot show.

## An agenda item that could not be settled, and should not have been

One run's headline item was who owned a piece of infrastructure. It was not
answerable inside the file: it turned on the operator's preference, and no
session held that. Both seats knew it from the header.

Carrying it was still right. What left the room was a documented recommendation
with the reasoning and each seat's position attached, and the user's answer
afterwards was four words. A settlement was never available; the preparation was,
and it was worth the run.

**But that is a different product, and the agenda did not say which one it
wanted.** An earlier run, on the same ambiguity, ended with a session closing an
item on the user's behalf -- which is the same failure the impersonation
prohibition addresses, arriving from the other end. The prohibition catches the
act of speaking as the user. Marking the item removes the reason to: a session
that knows at open that an item is `prepare-for-user` has no closure to reach
for, because closure was never the goal.

The cost is one word per agenda line, written by whoever creates the file, before
anyone joins.

The close-out rule is the half that makes it real. Reporting a
`prepare-for-user` item as *unresolved* is the same error as closing it -- both
describe it as a settlement that did not happen, and a session summarising toward
settlement is exactly how the user's decision gets made for them. It reports the
recommendation and where the seats differ, and it does not report a verdict.

Rejected: inferring the kind from the wording of the item. It is the same defect
the carry trigger was fixed for one version earlier -- a condition each seat
evaluates privately is one two seats will evaluate differently, and this one
would be evaluated silently by whoever wrote the close-out.

**An unmarked item defaults to `prepare-for-user`, which the rule did not say at
first.** As written, an omitted marker and a `settle` marker produced the same
artifact, and a close-out writer reading an unmarked item would take it for the
common kind -- reinstating the failure the rule was written to prevent, in the
one case where nobody had thought about it. This is the same shape as the carry
rule needing `carrying: nothing` rather than an omitted line: absence has to mean
something chosen, not something assumed. The default goes to the safe side, since
an unnecessary recommendation costs the user one line to wave through, while an
unnecessary settlement costs them a decision they never got to make. It also
keeps bridge files written before this rule from being read as all-`settle`.

## Carrying work out of the bridge (reasoned, not observed)

Every other rule in this file was paid for by a run that went wrong. This one was
not. It is marked so a later reader can weigh it accordingly, and so nobody cites
it as evidence it is not.

The close-out records conclusions; the DONE line records what evidence leaves the
room. Nothing recorded what **work** leaves the room, who holds it, or what it
waits on. So a run could end with an accurate close-out and no trace of the fact
that one repo's change does nothing until another repo's lands first -- and a
cross-repo dependency is exactly the kind of fact no single session can see,
because it exists only in the join. That makes it among the most valuable things
a run can produce, and the protocol had nowhere to put it.

Two observations argue for the rule without quite being incidents.

The first is already in this file: a finding that had sat fifteen days unmerged
was named twice in one run as an example, and never once picked up as work.
Agreement was reached and nobody left holding it.

The second is about how the protocol has been getting away with the gap. On one
run, a correction settled in a bridge did reach the repo that needed it, the same
day. The protocol contributed nothing to that -- the operator noticed and routed
it by hand. Every run so far has had one human holding both ends, which the
README already names as the untested condition, and a propagation step that works
only under that condition is the first thing to break when it stops holding.

The rule is deliberately about **order rather than dates**. A session cannot
commit its user's calendar: a date it offers is a promise about a person who was
never asked, made by something that will not exist when the date arrives. It is
also unverifiable from inside the file, which is the same objection this protocol
already raises against claiming delivery or readership. Ordering is different in
kind -- "B is inert until A lands" is a property of the work, checkable from both
ends, and it outlives whoever noticed it. Dates are admitted only as external
facts, a freeze or a meeting or a vendor cutoff, because those are observations
about the world rather than commitments by a participant.

Two refinements came out of review before this merged. Both are about the shape
of the rule rather than its substance, and both are worth recording.

The carry line was originally skippable whenever the work landed in one place.
That made an omitted line and "nothing to carry" indistinguishable to whoever
assembles the running order -- the same asymmetry the close-out rule already
names, where silence reads as consensus. And the condition was one each seat
evaluated privately, so two seats could disagree about whether the line was owed
at all, which is precisely the defect the round cap had just been fixed for. The
trigger is now observable from the file, more than one repo named anywhere in it,
and the nothing case is written rather than implied.

The second refinement bounds DONE itself. DONE has been accreting: it marks the
end of a contribution, inventories the evidence leaving the room, and now names
carried work and its ordering. Each addition was justified on its own and nothing
was watching the total, which is how a rule set becomes the very schema this
section rejects. The bound is that DONE carries only what cannot be reconstructed
from the file by a later reader. Both existing additions pass it, which is the
point -- it states a boundary the content already respects rather than
retrofitting a constraint onto it.

A third refinement, from the same review, and the one worth naming as a shape
rather than as a bug. The trigger is observable, but it is not stable in time. A
session that posts DONE while the file names one repo writes no carry line and is
right not to; if a second repo is first named later, that DONE is retroactively
non-compliant with a rule it satisfied when written, and its author may never
read the file again. The close-out then sees a DONE with no carry line and cannot
tell it from a seat that genuinely had nothing to carry -- the same asymmetry
`carrying: nothing` had just closed, re-entering through time instead of through
omission.

The fix sits on the close-out rather than on DONE. The close-out already reads
the whole file, and the alternative -- re-JOIN, which the protocol permits --
would make a session's exit conditional on entries not yet written. A DONE
predating the first mention of a second repo therefore has no carry status: an
open item, not a nothing, cited by both entry numbers. It fails in the safe
direction, since the worst case is a close-out naming an open item that turns out
to be empty.

Three findings of one shape in a single review is the useful part, more than any
of the three fixes. Each was a check that could not distinguish two cases with
opposite consequences, and each looked correct in isolation. That is worth
carrying forward as a question to ask of any new rule here -- what two situations
does this check collapse, and do they differ in consequence? -- rather than as
three separate lessons.

Rejected while writing this: an owner-and-status table in the close-out. It would
make the close-out authoritative about other sessions' commitments, which is what
the citation rule exists to prevent, and a schema invites being filled in rather
than meant. The commitment is self-attributed at DONE for the same reason a
position is cited rather than summarised.

### Then it ran, and the inputs were produced while the output was not

The section above is no longer unobserved. A three-party run under the carry rule
produced every input the rule asks for and none of the artifact it exists to
produce.

All three DONE entries carried `carrying:`, `waits on:` and what breaks if the
order inverts. The fields worked, and measurably: one seat's `waits on: nothing`
was contradicted by a peer who could see that a test proving a probe capable of
failing can only run while the thing it probes still exists -- a dependency
inverted by the session that had spent an earlier entry insisting on exactly that
discipline for someone else's artifact. That correction exists only because the
field forced the question.

Then nobody assembled the running order. The user asked for it after close,
having reasonably expected it, and a participant built it afterwards from the
three DONE entries, citing each line to its source.

The structural cause is not carelessness, and it is worth stating precisely
because the rule reads correct in isolation. **The only artifact the protocol
assigns an owner is the close-out, and the close-out is posted when the round cap
fires -- which is before any DONE exists.** The close-out on this run was
composed with zero DONE entries on file. The duty was therefore attached to the
one moment in the protocol at which it could not possibly be discharged, and
everything downstream of that moment was unowned.

One seat also had the evidence in hand and did not act. It corrected its own
inverted `waits on:` line at the exact moment it was holding two DONE entries and
reading the third, and it fixed its line rather than assembling the order. The
field made the author notice an error in its own row and did not make anyone
notice a missing artifact. A rule that produces inputs does not thereby produce
the output.

STOP is the only event this protocol defines by the inputs existing: every JOINED
session has DONE. That makes the closer the only participant for whom the duty
and the evidence arrive together, which is why the duty moved there rather than
to a new field or an earlier close-out.

Two amendments came out of the same discussion and both are about failure
direction. The duty attaches to the **close condition**, not to the STOP file,
because the second close path -- cap fired plus quiet, seats still un-DONE --
would otherwise ask a session for an order assembled from DONE entries that do
not exist; it instead owes what is on file plus the names of the seats missing
from it, since a named gap is a handoff and a silent one reads as nothing
pending. And every line is cited to the DONE it came from, with each seat asked
to correct its own rows, because without that the duty re-creates the exact
failure the close-out citation rule already guards against -- one session
summarising the others from impression -- and it re-creates it in the hands of
whoever happened to close, who is often the seat with the least stake in the
work.

### The durability instruction had no channel for a constraint

The same run exposed the adjacent gap. "Put whatever you agreed to carry
somewhere in your own repo" is right for a commitment, which only has to survive
where its owner will look. It says nothing about a **constraint on another
seat's** work, which has to reach a seat that does not read your repo.

A seat discharged it by writing into a peer's shared queue file, mid-run, while
that peer was editing the same file. Nothing went wrong, for two reasons that are
both accidents of this particular trio: that repo already carried a rule saying
peers may write there and must leave it uncommitted, and the peer ran a status
check before committing and found the foreign edit. Two of that session's commits
that night existed only to keep that clean.

A run whose participants share no such file has no defined durability channel at
all. The fix keeps writes inside the repo that owns them and moves the delivery
into the channel, which is the one surface every seat is already required to read
to the end: the carrying session writes it in its own repo and names the
constraint in its DONE, by entry number, addressed to the seat it constrains.


## One defect, six times, in five costumes

The rule under "Before you propose a rule" -- in `CONTRIBUTING.md`, since it
governs amendments rather than runs -- is stated once and abstractly. This is
the evidence, because the abstraction is not persuasive without it: every
instance below looked correct to the person who wrote it, and every one was
caught by somebody else.

| # | Amendment | What it collapsed |
|---|---|---|
| 1 | 0.6.0 carry rule, round 1 | The rule was skippable, so **an omitted line and "nothing to carry" read identically** to the consumer. |
| 2 | 0.6.0 carry rule, round 2 | The trigger was **a condition each party evaluated privately**, so two seats could disagree that it applied and both look right. |
| 3 | 0.6.0 carry rule, round 3 | The trigger became observable but **not stable in time** -- evaluated at write, in a file that keeps growing. |
| 4 | 0.7.0, stated counts | The count **enumerated the dividend and left the divisor bare**, so a missed JOINED or DONE could never show up in the ID list. |
| 5 | 0.7.0, agenda markers | **An unmarked item and a `settle` marker were identical** to the close-out writer, and the natural reading was the dangerous one. |
| 6 | 0.9.0, the read rule | The prescribed filter was `> high-water mark`, while **the same PR measured a quarter to a third of numbers shared** by two or more sessions. `> N` drops a peer's entry at your boundary number. |

Three things are worth drawing out.

**Rounds 1 to 3 are one defect arriving three ways, on one amendment, after two
fixes.** The author had already corrected the shape twice and did not recognise
it the third time, because it arrived through *time* rather than through
*omission*. Recognising a pattern in the abstract does not confer the ability to
spot its next costume.

**Number 6 is the one that should worry a reviewer most.** The PR stated the
correct rule -- `N` is the coverage key *once qualified by session* -- and then
dropped the qualification thirty lines later in the file where it mattered. The
sentence that falsifies the fix was in the same pull request as the fix. Nothing
about reading the PR carefully would have surfaced it; only checking the rule
against its own evidence did.

**The amendment that passed clean is the reason this is a rule and not a
complaint.** 0.8.0 hit the identical fork -- a close path where the inputs it
asks for do not exist -- and wrote the negative case unprompted, naming the
missing seats and what their absence leaves unknown. Same author, same week. The
check is learnable, which is what makes it worth the line it costs.

**What this section is not.** Six instances on one protocol, all found by review
between two workstations. Whether the up-front check actually fires before the
rule is written is untested -- every one of these was caught afterwards, by
somebody who did not write it.

**Postscript: two more arrived within a day of the table, both caught by an
outside reviewer rather than a run.** The merge-time fix for instance 6 replaced
`>` with `>=` -- a second numeric filter on the same broken premise, since a
slow seat's entry can land carrying a number below every threshold once faster
seats have moved on; the reviewer's scenario is now the reason the Loop tracks
`(session, N)` identities and prescribes no threshold at all. And the round
cap's close-out ownership turned on "unless the creator is visibly mid-reply"
-- a liveness judgement in a protocol whose own section heading says liveness
is not observable; ownership is now "first to observe the condition with no
close-out on file." Eight instances, no new costume: the second was Private
judgement again, the first was the same qualification failing to travel into a
*replacement* fix. A defect class does not stop applying to the corrections it
inspires.


## Protocol 2.0: why the transport changed and the file did not

On 2026-08-23 native cross-session messaging was verified working on this
platform -- including the property this design existed to protect (waiting
costs no model turns) plus one it never had (a message wakes an idle
session). The same verification, checked against the official documentation,
confirmed what the native channel lacks: a shared transcript, broadcast, and
any way to cross an OS-account boundary.

Protocol 2.0 is the consequence: the file remains the record and every rule
about it survives; the watch loop is replaced, on bridges that declare
`TRANSPORT: ping`, by a one-line pointer message to each other seat after
every append. The full design, its failure-mode analysis, and the review
that shaped it are in `2026-08-23-protocol-2-ping-transport.md` -- the spec
went through the three-pass review (self-critique plus two independent
reviewers on different models), which surfaced eleven distinct findings
before any of it touched the command. Three were found independently by both
reviewers: a STOP marker that no ping announces, a failure-record rule that
recursed into itself, and the quiet loss of the user's wake-the-room lever.
All eleven are addressed in the command; the ones that could not be fixed
(name reuse making delivery-to-a-stranger look like success) are stated as
prohibitions on inference instead, in the liveness section's new transport
clause.

Two follow-on amendments landed the same day, both operator-observed rather
than run-incident: session naming on ping bridges defaults to the address
itself. Earlier runs showed naming friction and ambiguity -- two names per
seat is a mapping that can drift, and hand-composed names collided or said
nothing -- while the address is unique among live sessions by construction
and, on repo-rooted sessions, already says what the seat is. The
evidence-descriptive naming flow survives as the watch-mode rule and the
ping-mode fallback for uninformative addresses. And bridge creation now
proposes ping unless a non-qualifying seat is expected, so the safer-by-
default header does not quietly become the recommendation.

Watch mode remains fully specified and is the default for any file without a
TRANSPORT line: it is the transport for cross-account bridges (the niche
native messaging cannot serve at all), downlevel clients, and harnesses
without messaging.

### The first ping-mode run

Four seats, one machine, sixteen entries, about ten minutes, run the same
day 0.12.1 shipped -- three seats executing the command text cold, the
fourth being the session that wrote it (noted in its own DONE as the least
independent reader in the room). What it established:

- **Every wake on every seat was ping-driven; no seat ran a poll or was
  pushed toward one by the text.** Confirmed first-hand from all four seats,
  each citing its own observation.
- **The close reached the room in order** -- final entry, pings, then STOP
  -- confirmed by independent receipt from every non-author seat, two of
  them citing the same STOP mtime (one marker, not two).
- **The post-append re-grep paid for itself repeatedly**: a three-way JOINED
  collision on one number, plus two entries landing inside the close
  author's own compose windows, all surfaced by the re-grep and absorbed by
  session-qualified citation.
- **Two seats posted simultaneous close-outs** on the same number -- the
  double-close race the ownerless close-out rule prices as acceptable. The
  two agreed on every item, which is the cheap side of the trade arriving
  as a free cross-check.
- **The post-STOP correction path worked in anger**: three additions landed
  after STOP -- an erratum, its root cause, and independent close receipts
  -- and every one reached every seat by ping. This is the exact 1.x
  incident class (a retraction appended into a dead room) the transport
  change existed to fix.
- **One real defect shipped and was caught by the run, not the review: the
  user-facing Post: line.** Two seats hit it independently -- bash eats the
  backtick-n before PowerShell runs, unexpected EOF, entry never posted --
  and the seat that proposed the fix then had its own example line mangled
  by the same nested-quote trap it was describing, which is why the shipped
  amendment derives the line in prose and instructs typing it fresh rather
  than copying any rendered version. Neither the spec's three-pass review
  nor its author touched this: the author seat never executes that line,
  only cold seats do. A reviewed line nobody executes is an example nobody
  executed.

Shakedown status after the run: burst behavior at four seats, the ordered
close, and post-STOP propagation are observed; the user nudge was exercised
in miniature (the user's go-check broke the deliberate quiet stretch);
**sender-side visibility of a held ping remains unexercised** -- no hold
occurred, every seat's permission class being compatible on this fleet --
so F3's in-file mitigation is still unverified, not passed.

Two observations from the seats' recaps, recorded but not yet legislated:
the close phase ran longer than the substantive phase (three near-identical
receipt confirmations, each with its own ping round) and that pile-up is
invisible to the round cap, which correctly excludes close-outs -- watch
whether it recurs on runs whose agenda does not invite receipts, and only
then consider a rule. And a seat renumbered its drafted entry after every
wake, work the protocol never asked for -- that one was cheap to fix and is
now an explicit do-not in the command.

### The second run, same day: the value proposition, exercised

The shakedown's sharpest recap refused to be satisfied: every finding in it
was corroborated, none corrected, because four seats doing the identical
exercise hold no divergent evidence -- "I would not read 'this went
smoothly' as evidence the channel is worth its cost on a real problem. That
test still has not been run." Ninety minutes later it ran: a four-seat
bridge over real migration-readiness state, three repo seats holding
genuinely different ground truth plus one observer seat holding only the
cross-repo handoff queue.

What the room did to its own claims, all by citation, inside an hour: a
seat corrected its own earlier framing after reading the actual docs (a
claimed cross-repo debt of "18 rows" turned out to be QA work misread as a
deliverable); the seat that had asserted that debt then re-verified
independently and retracted its own entry, killing a phantom dependency; a
filter result contradicted everyone's hunch (zero of 42 candidate files
were archived -- the seat validated its instrument in both directions and
disclosed a discarded false first run); and a content sweep tripled the
believed scope of a security finding, with the widening surfaced to the
user rather than closed in the room. Six-plus corrections, two of them
seats retracting themselves, none catchable by the claim's author.

Two patterns from that run worth naming. **The observer seat worked**: a
session with no repo stake joined carrying one verified piece of evidence
(an un-ingested cross-repo handoff both repo seats had standing to miss),
posted it, went DONE early so the round-cap divisor tracked the real
conversation, and stayed reachable -- its one flag (two seats offering the
same diff) was resolved by the seats within minutes. And **the ordering
constraint survived its authors**: the run's one real cross-seat dependency
was recorded in each owning repo before STOP, with the inversion cost
stated, so the close-out could cite it instead of inventing it.

Same-fleet caveat as everywhere in this file: one operator steering all
seats, every correction verifiable by one person out of band.

## Known caveats

- **The loop is not eternal.** It runs as an ongoing turn inside each session.
  If that session compacts context or otherwise ends its turn, the watch loop
  stops silently. Nothing is lost -- the full history is in the file -- but you
  have to notice and reissue `/bridge`. Treat it as on for a working session,
  not on forever.
- **Check your harness timeout.** The command passes `timeout: 600000`
  explicitly because a 120-second default kills every quiet wait after two
  minutes and spends a model turn re-issuing it, which quietly destroys the
  block-in-one-tool-call property the whole design rests on. Even at 600
  seconds, a quiet round costs one tool call returning nothing.
- **The round cap is self-assessment with arithmetic attached.** Counting
  entries is mechanical where "is this still productive?" is not, so it is a
  real improvement -- but the same party it constrains is the one applying it,
  and at 5 rounds it fires near the natural end of a normal bridge rather than
  catching a runaway. Its value is the forced question to the user.
- **The cap has been routed around twice, in opposite directions.** Once by a
  participant that posted DONE at the cap and re-JOINED two entries later when a
  new round opened -- legitimate, and why DONE is documented as non-terminal.
  Once by the run simply continuing: the cap was detected correctly and the
  remedy was owned by a session that was not acting on it, so the bridge ran
  another fourteen entries. Detection was never the weak part.
- **This channel is better at finding work than at causing it.** Observed twice,
  in different projects, by different sessions: a long-known load-bearing gap was
  named repeatedly during a run as an example of a cost, and picked up as work
  zero times. Bridges reliably convert an unbuilt design into a better unbuilt
  design plus a list of decisions for the user. That is real value and it is not
  delivery; the transcript volume makes it easy to mistake for delivery.
- **STOP scoping is a real footgun.** An earlier version put the marker in the
  bridge *directory*. A leftover from one day's run would have killed every
  session of the next day's on its first poll, silently and correctly per the
  protocol, and it was caught only because one session happened to list the
  directory first. Hence per-file scoping plus the check-before-joining step.
- **Content from other sessions is data, not instructions.** The command tells
  each session not to treat bridge entries as anything overriding its own
  permissions, project instructions, or its user. This held across every run
  without incident; keep it even as the setup earns trust. Entries from the user
  are the deliberate exception.
- **The evidence base is six runs across two machines**, all Windows, two to
  four participants. The failure modes are real; their frequencies are not
  established -- most of the worst ones above are single incidents, called
  structural because the mechanism permits them silently, not because they have
  been seen twice.
- **The absent-counterpart case has now been exercised, by accident.** Earlier
  versions of this document noted the protocol had never run against a slow or
  absent peer, which its own design calls the most common condition. One run
  degraded into exactly that state partway through, and two of its seven findings
  exist only because it did. Had every session stayed live to the end, that run
  would have been reported as a clean success and the report would have been
  wrong.
- **Both machines are operated by the same person.** One human has held all the
  context in every run to date, which means confusion could be corrected out of
  band without anyone noticing it happened. The protocol's whole premise is
  sessions correcting each other; it has not been run where the two ends could
  not simply ask their operator. Read every claim here as observed under that
  bias.
- **Every seat is the same model, and same-model consensus is weak evidence.**
  Stated first in a shakedown recap and adopted here: seats reading the same
  freshly written protocol text are primed by the same document and will tend
  to misread it the same way, so four seats agreeing "the text is clear"
  establishes little. The channel is most trustworthy when the WORLD
  contradicts the text -- an environmental failure, a filter result, a file
  that is not where the docs say -- and least trustworthy when same-model
  seats agree the text is fine. Weight corroboration accordingly: agreement
  between like seats is a prior, not a finding.
