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
- **The evidence base is five runs across two machines**, all Windows, two to
  four participants, every one with a live and fast-replying counterpart. The
  failure modes are real; their frequencies are not established. Never exercised
  against a slow or absent counterpart, which is the condition the design itself
  calls the most common one.
