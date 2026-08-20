# Review: what this tool is becoming, and a proposal for 1.0

**Status: proposal, not adopted.** Written 2026-08-19 after the 0.10.0 merges, at
the maintainer's request, to be picked up when time permits. Nothing here changes
behavior until it is decided on. The trigger was a failed trim: a plan to cut
`commands/bridge.md` from 575 lines to ~430 assumed the growth was narrative, and
the attempt proved it is not -- 112 of 218 lines of six-day growth were new
rules, each individually justified by a real incident. The question underneath
the trim is not "how do we make the file shorter." It is "what is this tool now,
and is its current form the right one."

---

## Evidence nobody asked for: the tool's own directory

At review time, one machine's bridge directory held:

- a STOP marker dropped **six days earlier**, never archived -- because archival
  runs "whenever a session next runs discovery," and no session on that machine
  had run `/bridge` since;
- two bridge files with **no STOP at all**: formally open, actually abandoned
  for a week.

That second state is the exact failure `protocol.md` names -- *"the bridge sits
formally open while actually abandoned, which reads from inside as still
live"* -- happening silently, in production, undetected by the protocol that
predicted it.

That is the diagnosis in miniature: **the protocol only executes when the tool
happens to run, and the tool runs rarely.** Meanwhile the protocol grew 38% in
six days. Maintenance has inverted over use: five runs total, none on that
machine in six days, and roughly five evenings of protocol work in the same
window.

## The empirical split that decides the architecture question

Sort everything in this repo by *who executes it* and the record is unambiguous:

- **What the script executes** (the blocking watch, PID discipline, STOP
  detection): zero failures since 0.3. The "Confirmed properties" section of
  `protocol.md` is entirely about this layer, and it was tested adversarially.
- **What the model executes mechanically** (whole-file reads, entry counts,
  divisors, high-water marks, coverage): **all six shipped defects**, including
  one rule broken twice -- the second time by a session that had read the fix
  and could quote it.
- **What the model executes as judgment** (the impersonation prohibition,
  citation discipline, `settle` vs `prepare-for-user` semantics, the round-cap
  question priced by the user): held up well, and produced the three mechanism
  corrections that justify the tool's existence.

`protocol.md` already contains the epitaph for the middle layer: *"a rule that
competes with an affordance loses."* The protocol holds the proof that it cannot
win that fight, and keeps writing more of the rules that lose.

Reading the command end to end, roughly **half** its rule mass sits in that
middle layer. That is a read-based estimate, not a tally -- but it is not close
to a third.

## What this has actually become: three things wearing one name

**1. A transport.** File-as-bus, blocking wait, no daemon, no auth. Commodity,
and deliberately so. Its one killer property -- **waiting costs zero model
turns** -- must be defended at all costs, but nothing else in this layer is
precious. The moment native peer-session messaging works on this platform, this
layer is obsolete on purpose, and that should be a planned retirement rather
than a surprise.

**2. A review ritual.** This is the durable intellectual property, and it is
transport-independent: convene seats holding *different* evidence, an agenda
with `settle` vs `prepare-for-user`, adversarial correction, cite-or-report-as-
not-established, a close-out priced by the user rather than self-assessed.
Every documented win came from this layer. Notice the tool never won as a
*messaging* system -- routine fact-passing between sessions is better served by
direct writes to a shared store. The proven niche is narrow: **claims one seat
cannot check alone.** It should stay that narrow.

**3. A research notebook.** `docs/protocol.md` is quietly the best artifact in
the repo -- rejected designs, adversarially tested properties, honest cost
accounting, a six-instance defect taxonomy. It is original empirical material on
multi-agent coordination discipline. It does not need trimming; it needs to keep
being what it is.

The trouble is that all three currently live behind one 777-line prompt that is
loaded, and probabilistically obeyed, on every invocation. Rules that fire
rarely -- the impersonation prohibition above all -- compete with 113 bolded
siblings for a reader's attention.

## The proposal: call it 1.0 and consolidate; stop growing

### 1. Ship a script with verbs; shrink the prompt to judgment

The plugin already ships an inline watch script. Promoting it to
`scripts/bridge.ps1` with `read` / `status` / `append` / `close` / `archive`
verbs is the same move at full size. What each shipped defect class becomes:

| Defect (shipped) | Script affordance that makes it unrepresentable |
|---|---|
| Anchored slice / high-water filter (3 instances) | `bridge read`: a per-session cursor stored as a **byte offset** into the append-only file. Offsets are monotonic, so sequence-number collision semantics stop mattering for reading entirely. The cursor is written only *after* printing; a missing or corrupt cursor falls back to dumping the whole file. Fails loud, and toward showing more. |
| Bare divisor, disputed counts | `bridge status`: entries, IDs, live seats, rounds -- one arithmetic authority. Two seats cannot disagree about a number neither of them computes. |
| Omitted `carrying:` line, unmarked agenda items | `bridge append -done` emits the entry skeleton with the mandatory fields already present. Absence now requires deleting a line, which is a choice, not an oversight. |
| STOP never archived | Any verb sweeps for stale STOPs on invocation, not only discovery. |

The command then drops to somewhere near 300 lines, all judgment: identity,
citation, agenda semantics, carry-and-constraint semantics, when to use this at
all. The rules that fire rarely stop being buried.

**Collapse test applied to this proposal itself** (per "Before you propose a
rule"): a script the model *trusts* replaces six rules with one failure point.
So the script must fail loud -- corrupt cursor means full dump, never a silent
slice -- and the model keeps exactly one cross-check: the close-out still cites
entry IDs from its own read of the file. The belt stays; the six suspenders go.

### 2. Freeze growth with the protocol's own rule

The DONE-budget insight already in the command -- *"every addition has been
defensible on its own and nothing has been watching the total"* -- applies to
the protocol itself, which grew a rule every seven lines and 38% in six days
while every single addition was individually justified. So:

- Give `commands/bridge.md` a hard line budget. A new rule displaces an old one
  or becomes a script affordance. A constitution with an amendment cost.
- Demote n=1 rules honestly. Several merged PR bodies admit "one instance, not
  a trial" or "reasoned, not observed." Law goes in the command; case history
  goes in `protocol.md` until it recurs. The rule count today does not discount
  for evidence strength, and it should.

### 3. Optional hardening

- Encode the anchored-slice regression as a plugin eval, so the failure that
  shipped twice cannot ship a third time silently.
- Re-check periodically whether native peer messaging has landed on this
  platform. When it does, the transport retires and the ritual survives it.

## Angles worth keeping in view

- **The PR loop is a second product.** Five PRs between two machines that share
  no session, four catching real defects in both directions, review-by-stranger
  enforced by fork mechanics. That practice ports to anything the two machines
  co-maintain. It was discovered here, not designed.
- **The ritual is portable off this transport.** Written cleanly, the review
  discipline could run over native messaging the day it exists -- or over a
  deliberately convened pair where one seat is spun up cold purely to refute.
  Keeping ritual and transport fused in one document is a liability.
- **`protocol.md` may be worth surfacing on its own terms**, as a field
  notebook on making LLM sessions coordinate honestly. The repo is already
  public and anonymized; a README pointer framing it as findings costs nothing.
- **The single-operator caveat stays load-bearing.** All evidence comes from
  one person steering both ends. The protocol may be overfit to one operator's
  habits, and 1.0 should not claim otherwise.

## What not to do

- **Do not rewrite as an MCP server or daemon.** It kills self-configuration
  and risks the free-wait property. A related project died precisely because
  each empty poll cost a full model turn.
- **Do not build addressee filtering.** Already rejected with evidence: the
  entries that most need a reply are addressed to someone else.
- **Do not aim at autonomous or unattended operation.** The entire safety model
  assumes the user is in the room.

## Options

1. **Consolidate to 1.0 as above** -- script for mechanics, ~300-line judgment
   prompt, growth freeze -- and send it through the PR loop so the process
   reviews its own restructuring. *Recommended.*
2. Leave the structure as-is and enforce only the growth freeze. Cheapest, but
   the six-defect layer stays load-bearing.
3. Retire the transport now and keep the ritual as a documented practice.
   Premature: the free-wait property has no native substitute on this platform
   yet.

Timing note, recorded so momentum does not decide it: this is an evening of
real work, nothing about it rots if it waits, and the tool has more pressing
company. Schedule it; do not let the enthusiasm of the review pick the date.
