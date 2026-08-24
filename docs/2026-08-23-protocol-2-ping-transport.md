# Protocol 2.0: the file stays the record, SendMessage becomes the doorbell

**Status: ADOPTED 2026-08-24, implemented as protocol 2.0 / plugin 0.12.0
the same day.** Three-pass review complete before adoption: pass 2
(self-critique) and pass 3 (independent dual review: fresh-context Claude +
Codex) both folded in. Pass 3 surfaced 4 confirmed-by-overlap findings and 6
unique ones; all accepted, two with pushback recorded in section 12. One
routing deviation from section 13 as written: the command rewrite went to
main under maintainer custom rather than waiting on the cross-machine PR
loop -- the three-pass review supplied the independent scrutiny, and the
fork loop reviews post-merge as it has before. The shakedown tests in
section 13 remain open until a real multi-seat run executes them.**

Written 2026-08-23, the same day native cross-session messaging was verified
working on the Windows CLI (see the disposition note in
`2026-08-19-review-what-this-is-becoming.md`). This spec is the answer to the
question that disposition left open: whether the ritual ports onto native
messaging or stays on the watch loop. The answer proposed here is neither --
the ritual stays on the file, and native messaging replaces only the one thing
the watch loop ever provided: waking the other seats.

---

## 1. The one-sentence design

Every rule about the file is unchanged; the watch loop is replaced by a
one-line `SendMessage` ping sent to each other seat after every append, and a
seat that has nothing to say simply ends its turn instead of blocking in a
poll.

## 2. Why

The 1.0 review sorted the protocol into three layers and found the ritual
(agenda, citation, close-out, carry) is the durable intellectual property
while the transport is deliberately commodity. Tonight's verification
established that the native channel now has the transport's one killer
property -- waiting costs zero model turns -- **plus one the watch loop never
had: it wakes an idle session.** What the native channel lacks, confirmed
against the official documentation rather than assumption, is exactly what
the file provides: a shared transcript, broadcast, and an account boundary
crossing.

So the two halves are complementary, not competing:

| Property | File (1.x) | Native messaging | 2.0 hybrid |
|---|---|---|---|
| Shared, citable, auditable record | yes | no | yes (file) |
| User watches one window, posts in | yes | no | yes (file) |
| 3+ seats, one provable record | yes | no | yes (file) |
| Waiting costs zero turns | yes, via blocked tool call | yes, via idle | yes (idle) |
| Wakes an idle seat | **no** | yes | yes (ping) |
| Seat can do other work between rounds | **no** | yes | yes |
| Reaches a seat after STOP | **no** | yes | yes (ping) |
| Crosses OS-user accounts | yes | no | yes (watch-mode fallback, section 8) |

The two boldest wins are behavioral, not mechanical:

- **The bridge stops monopolizing sessions.** Under 1.x, a three-seat bridge
  is three sessions blocked in watch loops for the duration -- the cost the
  protocol's own honest-accounting section keeps apologizing for. Under 2.0 a
  seat answers, goes back to its real work or goes idle, and is woken for the
  next round.
- **The empty room stops being silent.** Three documented 1.x incidents --
  the correction appended after STOP that nobody read, entries addressed to
  seats that had exited, the recap written from an unfinished read -- all
  share one mechanism: a write can only reach a seat sitting in the watch.
  Under 2.0 the write's ping reaches the seat wherever it is.

## 3. Terms

- **Ping**: a one-line `SendMessage` from one seat to one other seat, carrying
  a pointer to a new entry and never the entry's content.
- **Address**: the name a session answers to in `ListAgents` (e.g.
  `claude-2a`). Distinct from the bridge session name, which describes
  evidence held, not location.
- **Ping mode is same-machine, same-account by construction.** The bridge
  file lives on a local fixed disk (network drives are banned by the 1.x
  drive rules, unchanged), so every seat shares one machine -- and every
  native-channel behavior this spec relies on (sender-side hold notices,
  burst caps, delivery semantics) is the documented same-machine case. No
  part of this design involves Remote Control or cross-machine delivery.
- **Ping mode / watch mode**: the bridge's transport, declared once in the
  file header (section 8). All seats use the declared transport; mixing is
  forbidden.

## 4. Changes to joining

The JOINED entry gains an address field:

    ### [001] | from: kb-west | JOINED | address: claude-2a

- The address is this session's own first line from `ListAgents` (or
  `/list-agents`). Look it up at join time; do not recall it. **The practical
  version floor for ping mode is Claude Code v2.1.239**, not the v2.1.234
  messaging floor: before .239 the listing does not show the session's own
  name, and the join step cannot be executed as written.
- **After appending JOINED (or any entry), re-grep the headers once more.**
  The re-read-before-append rule leaves a window: an entry that lands while
  you are composing is in nobody's ping set -- two simultaneous late joiners
  can each see only the older seats and miss each other, and a roster gap
  during a quiet stretch has no ping to heal it. The post-append re-grep
  costs one cheap read and closes the window to milliseconds; anything it
  reveals is processed as a normal wake.
- **If your address changes** (session restart, rename), append a one-line
  entry: `### [N] | from: kb-west | ADDRESS | now: claude-7f` and ping all
  seats from the new address. A stale address fails loud at the sender --
  `SendMessage` errors on an unknown name -- which is a better failure than
  any the watch loop had, and the sender's recovery is to re-read the file
  for an ADDRESS entry before concluding the seat is gone.
- **Transport entries do not count toward the round cap and are never
  pinged.** Transport entries are: ADDRESS (above), the downgrade entry
  (section 8), and delivery-failure records (section 5). The 1.x counted-
  entries subtraction ("every entry except JOINED, DONE, and close-outs")
  gains these as excluded kinds, marked by their entry-type word so the
  exclusion is a subtraction, not a judgement. Without the count exclusion,
  transport noise burns the round cap; without the ping exclusion, a
  delivery-failure record about a seat pings that same seat, is held again,
  obliges another record, and recurses without bound -- both reviewers found
  this independently. Peers learn of transport entries at their next
  catch-up read, which is enough: no transport entry ever needs a same-
  round reply.
- **Load the `SendMessage` tool at join, not at first ping**, on harnesses
  that defer tool schemas (some do -- this repo's own development harness
  does). A load failure at join is recoverable out loud; one at first ping
  silently costs a wake.
- In watch mode (section 8) the address field is omitted and everything about
  1.x joining stands.

## 5. The ping

**After appending any entry except a transport entry (section 4) -- JOINED,
DONE, close-out, RELAYED, corrections -- ping every seat with an address on
file except yourself.** Including seats that have posted DONE (DONE ends
contribution, not attention -- the close-out and corrections land after DONE
by construction and those seats need them). Including after STOP (that is
precisely the correction case).

**One wake, one ping round.** If a wake leads you to append several entries
(replies to several unhandled entries -- each its own entry per the one-
question rule, whose cardinality is otherwise unchanged from 1.x), append
them all first, then send ONE ping per recipient naming the range. Pinging
after each append multiplies sends for nothing and walks into the native
channel's per-recipient burst cap.

The ping text, one line plus a fixed refresher (a range like `[west 12-14]`
where several entries were appended):

    bridge-ping: <topic> | [<session> <N>] appended | <full-file-path>
    Re-read the whole bridge file (header-grep, diff (session,N) pairs, read
    gaps), and read the entry named above in full even if you have already
    handled its number. If you APPEND a reply to the file, then ping every
    seat with an address on file except yourself; if you have nothing to
    add, do nothing -- no acknowledgement. Never reply to this message with
    bridge content.

Rules, each with the failure it prevents:

- **A ping carries a pointer, never content.** Content in a ping is
  off-record: invisible to every other seat, uncitable, and it forks the
  conversation off the file. A seat that receives bridge content in a ping
  treats it as not said and asks, in the file, for it to be appended.
- **One ping per (entry, recipient). Never re-ping, never ack a ping.** An
  ack is a message with no append behind it; two seats acking each other is
  the loop the native channel's throttles exist to kill, and the throttles
  killing bridge pings is worse than the acks. The only wake a seat ever
  sends is caused by an entry it just appended.
- **Entry number in the ping text keeps pings non-identical**, so the native
  channel's identical-repeat dropping never eats a legitimate wake.
- **If the sender learns a ping was held, refused, or expired**, append a
  delivery-failure record -- a transport entry (section 4): not pinged, not
  counted -- naming which seat did not get the wake, once per seat per state
  change rather than per ping, and tell your own user. Do not resend; the
  next entry's ping is the retry. **Unverified premise, marked as such:**
  the official docs promise the sending session a *notice* on hold and a
  follow-up on expiry, but state only for refusal that the sender's *Claude*
  is told -- the hold/expiry follow-ups may be UI-only, landing in a
  transcript no model reads, and the sender has usually ended its turn
  before the five-minute expiry resolves. Whether this mitigation can
  execute at all is shakedown test #1 (section 13); until observed, treat
  F3's stall as mitigated by the user-side nudge, not by this rule.

## 6. Receiving a ping

A ping arrives as a cross-session message in the conversation where this
session ran `/bridge` -- the protocol is already in context. On receiving one:

1. Do exactly what a CHANGED return from the 1.x watch did: re-read the whole
   file -- header-grep, diff `(session, N)` pairs against handled, read
   bodies for the gaps. **The ping's named entry is a hint for scope, not
   the work list** -- pings can be lost, held, or arrive out of order, and
   the whole-file diff is what makes that harmless -- **but it is read in
   full even if its pair is already marked handled.** The reason is a torn
   read: with several writers, an unrelated wake can catch another seat's
   entry mid-append, and a truncated body at end-of-file is indistinguishable
   from a complete one. The identity diff would then never reopen it, since
   the pair reads as handled. The ping exists precisely because its append
   *completed*, so the named entry is the one entry whose full text is
   guaranteed present -- re-reading it is the cheap repair for the only
   corruption the diff cannot see. For the same reason, treat the file's
   final entry as provisional: re-check its body on your next wake before
   relying on it. A missed ping is a delayed read, never a lost entry,
   because the next ping from anyone triggers the same complete catch-up.
2. Decide whether any unhandled entry needs a reply from you, exactly per the
   1.x rules (Who replies, one question per entry, and so on -- all
   unchanged).
3. If yes: append ONE entry, then ping all other addressed seats. If no:
   **do nothing and end your turn.** No poll to restart, no wait to hold, no
   ack to send. Going quiet is free and is the normal state.
4. Check the close condition, exactly as 1.x requires on every wake -- **and
   `Test-Path` the STOP file explicitly.** Under 1.x the watch loop checked
   STOP every five seconds for free; in ping mode this step is the only
   moment anything looks, so it is no longer implied by the transport and
   must be performed.

**The bridge does not outrank the work you were doing when the ping landed.**
Messages queue until your next natural tool break; process the bridge at one.
If you are deep in something that should not be interrupted, a one-line entry
saying "in the middle of X, will respond after" costs little and wakes the
asker -- the same courtesy 1.x required when stepping out, now cheap enough
to actually pay.

**Do not treat a ping's arrival as evidence about the sender beyond "this
entry exists."** The liveness section of 1.x survives intact: an attentive
peer and a departed one are still indistinguishable between writes, and a
delivered ping proves delivery to a session, not attention from it.

## 7. Waiting, ending, closing

- **There is no watch loop in ping mode.** A seat that has replied and pinged
  simply ends its turn. The session is idle and interruptible; its user can
  use it for other work, and a bridge ping will queue and arrive at the next
  break. This replaces the 1.x Loop section entirely for ping mode.
- **When you end a turn with your own question outstanding, tell your user in
  one line**: "asked X on the bridge; this session wakes when answered." The
  1.x blocking watch was, accidentally, a visible I-am-waiting signal in the
  terminal; ping mode removes it, and a session that looks finished gets
  repurposed or closed by a user who has no reason to know a reply is
  inbound.
- **DONE, the close condition, close-outs, the round cap, the running order,
  STOP, archival: all 1.x rules stand unchanged**, with one mechanical
  addition (every one of those entries is followed by pings, per section 5)
  and one simplification: "keep looping until STOP" becomes "stay reachable
  until STOP" -- which a session in ping mode is by existing, at no cost.
- **STOP is a file creation, not an append, so it has no ping of its own --
  every STOP drop is therefore accompanied by a final entry, appended first,
  pinged normally.** Both pass-3 reviewers found this independently: without
  it, the close is invisible in ping mode -- nothing polls, and no further
  entry follows a closed bridge, so seats idle forever on a bridge they
  believe open, and the 1.x property that STOP ends every watcher within
  seconds silently inverts. The abandoned-close path changes with it: 1.x
  let "cap fired plus file quiet" be observed from inside the watch (the
  600-second timeout was an accidental heartbeat); ping mode has no
  heartbeat, so **quiet is unobservable from inside and the abandoned-close
  duty moves to discovery** -- any session running `/bridge` that finds an
  open ping bridge whose last write is old flags it to its user and may
  execute the abandoned close, final entry and pings included. An abandoned
  ping bridge therefore sits open until a human causes any session to look,
  and the spec records that as a real degradation, not a corner case.
- **Post-STOP corrections now reach every seat.** Append the correction,
  ping all addressed seats. The 1.x incident where a retraction landed after
  every watcher had exited cannot recur in ping mode. The re-read-before-
  durable-writes rule stays anyway; pings reduce its load-bearing weight but
  a held ping (F3) means it still earns its place.
- **A converged bridge needs no exit patience.** The 1.x "agreement is a
  reason to close" rule stands, and its cost drops: two seats that have
  agreed are not holding loops open while they wait to be sure; they are
  idle. The failure that rule was written for -- both seats polling into
  mutual politeness -- is structurally gone, but the rule stays because
  saying the agreement out loud is what lets the close condition be met.

## 8. Transport declaration and the watch-mode fallback

The creator declares the transport in the file header, at creation, next to
the AGENDA:

    TRANSPORT: ping        (must be declared explicitly)
    TRANSPORT: watch       (the 1.x watch loop, unchanged -- and the default
                            when the line is absent)

**Absence means watch, not ping.** Every bridge file created before 2.0 lacks
the line; if absence meant ping, a 2.0 seat joining a still-open 1.x bridge
would ping seats that are polling and wait for pings from seats that will
never send one -- the exact mixed-transport stall this section forbids,
arriving through backward compatibility. Absence-means-watch is safe in both
directions: the worst case is a creator who forgets the line and gets 1.x
behavior, which is degraded, not broken. Ping mode is always a choice
somebody wrote down.

- **Why declared, not inferred:** a bridge where some seats ping and some
  poll has a silent hole -- a watch seat's append wakes ping seats never
  (nobody polls) and a ping seat cannot ping an address-less seat. Both
  halves of a mixed bridge stall in a way neither can see. A declared
  transport is observable by every seat from the file on every read, which is
  the property the contributing rules require of any trigger. An inferred
  one (e.g. "ping if everyone has an address") flips when a late joiner
  arrives without an address, and a condition that flips over time on a
  growing file is disguise number three in the defect taxonomy.
- **Watch mode is the 1.x protocol verbatim** -- watch script, read-time
  baseline, timeout guidance, all of it. It remains the transport for: a
  seat under a different OS account (native messaging cannot cross that
  boundary at all -- the bridge's most durable niche), a downlevel Claude
  Code without messaging, any harness where `SendMessage` is absent or
  denied, and any future environment this repo has not seen.
- **A seat that cannot ping must not join a ping bridge.** If you cannot see
  `SendMessage` in your tools, or `/list-agents` is not recognized, say so to
  your user and either join in watch mode on a watch bridge or do not join.
  A creator who knows a cross-account or downlevel seat is expected declares
  `TRANSPORT: watch` up front.
- **The header is not a bilateral gate, and the spec does not pretend it is.**
  A 1.x client knows nothing of the TRANSPORT line: it can join a ping
  bridge and poll, hearing every write (its watch sees file mtime) while its
  own appends wake nobody. The mandated response is an explicit **downgrade
  entry**: the first ping-mode seat whose wake reveals a JOINED without an
  address appends `### [N] | from: <name> | DOWNGRADE | to: watch` (a
  transport entry: not counted, but **pinged, as the one exception** --
  every addressed seat must learn the transport died), after which every
  seat runs the 1.x watch loop and the bridge is watch-mode to its end.
  Downgrade is the only mid-run transport change; upgrade never happens
  mid-run. The trigger is observable (an address-less JOINED in a
  ping-declared file), the direction fails safe (watch works for everyone),
  and the window before some ping causes a 2.0 seat to notice is the same
  heals-at-next-entry window as the joiner race in section 4 -- during it
  the 1.x seat hears everything and the 2.0 seats owe it nothing they know
  about.

## 9. User participation

**The user's lever is materially weakened in ping mode, and this section says
so plainly rather than filing it under "unchanged mechanics."** Under 1.x a
direct user append woke every watcher within seconds -- protocol.md calls it
the only reliable way to unstick a stalled bridge, and a stalled ping bridge
is precisely one where no seat will append and therefore no ping will ever
carry the user's directive to anyone. The watching half survives; the
speaking half now needs a second action:

- The user still watches with `Get-Content -Wait` and still posts with
  `Add-Content`, exactly as 1.x. Both are file operations; the transport
  change does not touch them.
- **A user's append wakes nobody by itself** -- no ping behind it, no poll
  in front of it. Same for a user-dropped STOP, which is the user's kill
  switch: a file creation pings nobody. The answer is procedural: after
  posting an entry OR dropping STOP, nudge any one seat in its terminal
  ("check the bridge") -- that seat reads, and its resulting pings carry the
  news to the rest.
- **The nudge instruction lives in the bridge file header**, written by the
  creator at creation next to TRANSPORT, not only in a one-time terminal
  printout that is windows and possibly days away when the user needs it:

      USER: after posting or dropping STOP, tell any one session
      "check the bridge" -- your write wakes nobody by itself.

  The creator still prints the observe/post lines at creation as 1.x
  requires, with the nudge line added.
- Rejected for now: a script that posts wake-ups to each seat's inbox socket
  (the socket paths and per-session tokens make this buildable). It is the
  right shape if the nudge proves annoying in practice; it is machinery with
  an auth surface if built speculatively. Revisit on evidence.

## 10. Failure modes, each with its collapse test

Per CONTRIBUTING: what two situations does each mechanism fail to
distinguish, and do they differ in consequence?

- **F1. Forgotten or incomplete ping fan-out after an append** -- forgotten
  outright, or the appender crashed, was closed, or compacted mid-fan-out,
  reaching some seats and not others. The unreached peers stay unwoken until
  the next entry's ping triggers the catch-up read, and a partially woken
  room usually heals itself faster: any woken seat's reply pings everyone.
  Collapse: a forgotten ping and a deliberately unsent one (there is no
  deliberate case -- the duty is unconditional) -- no consequence gap.
  Degraded to latency by the whole-file read; the entry is never lost --
  **unless it is the final entry, which is F3's class regardless of why the
  ping went unsent.** This is the residual mechanical duty and the known
  weakest point: model-executed bookkeeping, the layer every 1.x defect
  lived in. The mitigation is its unconditionality plus the self-healing
  consequence; deliberately NOT a delivery ledger -- persistent transport
  state is where the 1.x defects lived, and a ledger that can be wrong is
  worse than a re-read that cannot.
- **F2. Ping without an append** (content in the ping, or an ack). Forks the
  record or feeds the throttles. Guarded by the never-content rule and the
  fixed refresher line in every ping telling the recipient to treat message
  content as off-record.
- **F3. The final entry's pings go unsent or undelivered -- held, dropped,
  refused, or the sender died mid-fan-out -- and no further entry ever
  comes.** The one real stall class: nothing self-heals a wake with no
  successor. Mitigations, in order of reliability: the user-side nudge (the
  user watching the file sees the entry nobody answered -- section 9's
  header line covers it); the sender's delivery-failure record and user
  report (section 5 -- **premise unverified**, shakedown test #1); and on a
  single-operator fleet, removing mixed-mode holds entirely with
  `crossSessionInbound: accept` in user settings.
- **F4. Stale or dead address -- and the failure is NOT reliably loud.** A
  renamed, restarted, or closed session usually makes the ping fail with an
  unknown-name error at the sender; recovery: re-read for an ADDRESS entry,
  and absent one, append a delivery-failure record. **But names are
  reusable**: once the original session has exited, an unrelated later
  session can answer to the same name (auto-generated names especially), and
  the native channel delivers on the name alone when exactly one session
  answers. The ping then *succeeds* -- to a stranger, who per F7 declines a
  bridge it never joined -- while the sender records a delivered wake. So
  the liveness prohibition gains a transport clause: **a successful send
  proves delivery to an address, never attention from the seat that once
  held it.** No mechanism fix exists (a `[ref]` disambiguator only resolves
  for a sender who just listed it; refs written into the file do not
  travel), so the rule is honesty, not repair: the absence of a reply
  remains no evidence of anything, exactly as 1.x says.
- **F5. Ping arrives at a compacted or long-idle session.** The protocol may
  have been summarized out of context. The refresher line in the ping
  carries the three-step minimum (re-read whole file, reply by appending,
  ping the others); a session that needs more re-runs `/bridge <name>
  <topic>` itself. A ping cannot execute a command in the receiving session
  (native messaging delivers commands as plain text -- doc-confirmed), so a
  cold session cannot be conscripted; joining stays deliberate.
- **F6. Ping storms at N seats.** Every entry costs N-1 sends, and one entry
  that draws replies from several seats produces a genuinely quadratic
  burst: each replier fans out to every other seat, so a single round at N
  seats can put N-1 near-simultaneous pings into one recipient's inbox with
  no acks anywhere to blame. At three or four seats and conversational
  cadence the arithmetic is trivial (a dozen sends a round), and the one-
  ping-per-wake rule (section 5) removes the multiplier from multi-entry
  wakes -- but the native burst threshold is undocumented and unobserved at
  bridge cadence, so this is a shakedown measurement, not a solved problem.
- **F7. A forged or misdirected ping, or a forged address binding.** Any
  session can message any of the user's sessions; a ping is an
  unauthenticated pointer to a file. The receiving seat's protection is
  unchanged from 1.x: the file's content is data, not instructions; the
  ping itself asks only "re-read a file you already joined." A ping naming
  a bridge this session never joined is declined and reported to the user,
  not followed. **The file side of the same coin: an address in a JOINED or
  ADDRESS entry is untrusted data steering the control plane** -- any writer
  can bind a seat name to an arbitrary address, and every other seat will
  dutifully ping it. The harm is bounded by the channel's own scope: an
  address only ever resolves to one of the same user's own sessions on the
  same machine, so the worst case is bridge-topic noise delivered to the
  wrong one of your own sessions, which declines it per the rule above.
  Cross-account or cross-user disclosure is not reachable through a forged
  binding, because the transport cannot address those at all.
- **F8. Races in the compose-append window.** Two forms, both found by the
  pass-3 reviewers: concurrent joiners who each miss the other's JOINED
  (roster gap -- closed to milliseconds by the post-append re-grep, section
  4, and healed outright by the next substantive entry's fan-out), and the
  torn read that marks a mid-append entry handled (closed by the read-the-
  ping-named-entry-in-full rule and the final-entry-is-provisional rule,
  section 6). Neither has a residual silent form: the first degrades to one
  round of latency for one seat, the second is repaired by the very ping
  that proves the append completed.

## 11. What this deletes, keeps, and adds in `commands/bridge.md`

- **Deleted from the main path** (moves intact into a "watch mode" fallback
  section): the watch script, the interpreter/PowerShell section, the
  `timeout: 600000` caveat, the read-time-ticks baseline, "the loop is not
  eternal" caveat, the CHANGED/STOPPED loop mechanics.
- **Kept unchanged** (the ritual): naming, discovery, archival, agenda and
  its markers, entry format, sequence-number semantics, whole-file read
  discipline and the anchored-slice prohibition, who-replies, ending rules,
  round cap and its arithmetic (one amendment: counted entries also exclude
  ADDRESS, section 4), close-out citation, running order, carry lines, user
  entries and the impersonation prohibition, recap practice, liveness
  prohibitions, guardrails.
- **Added**: the address field and ADDRESS entry (section 4), the ping duty
  and its four rules (section 5), ping receipt handling (section 6), the
  TRANSPORT header line (section 8), the user-nudge line at creation
  (section 9).
- Net effect on the loaded prompt: roughly neutral to modestly negative in
  lines -- the watch machinery leaves the hot path but ping rules enter it.
  This spec does NOT deliver the 1.0 review's ~300-line prompt; that remains
  the separate, still-on-hold script proposal, and nothing here blocks it.

## 12. Non-goals and rejected designs

- **Content over SendMessage.** Rejected above (F2); restated here so it is
  not rebuilt: the file is the record *because* every seat provably can read
  the same bytes; point-to-point content has no such property at any number
  of seats above two, and none at two the moment anyone cites.
- **`notify_when_idle` integration.** One-shot, same-machine, main-
  conversation-only, 12-hour expiry -- a poor fit for a multi-round
  conversation, and the ping already carries the wake. Revisit only if a
  real run produces a need.
- **Addressee filtering of pings** (ping only the seat a question is for).
  The 1.x rejected-designs section already killed filtering wakes by
  addressee, with evidence: the entries that most need a reply are addressed
  to someone else. Every entry pings every addressed seat.
- **Automatic transport fallback / mixed transports.** Rejected in section 8.
- **A daemon, a coordinator, a message broker.** Still dead; see 1.x
  rejected designs and the 1.0 review's do-not-build list.
- **A delivery ledger** (per-recipient sent/acked state to resume an
  interrupted fan-out). Proposed by pass-3 review against F1; rejected:
  persistent transport state is the layer every 1.x defect lived in, and a
  ledger that can be wrong is worse than a whole-file re-read that cannot.
  The self-healing read is the ledger.
- **Making the round cap tighter to absorb transport noise.** Rejected in
  favor of excluding transport entries from the count (section 4) -- the
  cap's value is that every seat derives the same number, and any exclusion
  must stay a subtraction by entry-type word, never a judgement about
  weight.

## 13. Versioning, rollout, evidence

- This is protocol 2.0.0: the largest amendment the repo has taken. Plugin
  version 0.12.0 on adoption.
- Route: through the PR loop per repo custom (fork review from the second
  machine), given the size. The spec itself is pass-1 of a three-pass
  review; the command rewrite happens only after the spec survives all
  three.
- **Shakedown on a real need, not a synthetic run**: the operator has
  multiple pending 3-seat coordination cases. First real run is the test;
  the recap practice (unchanged from 1.x) captures what broke. Named
  shakedown tests, in priority order:
  1. **Does a sender's Claude ever learn of a held or expired ping?** F3's
     in-file mitigation rests on it and the docs do not promise it. Test by
     sending from a prompting-mode seat to a bypass-mode seat and letting
     the hold expire.
  2. **Burst behavior at 3 seats**: several repliers fanning out in one
     round -- do any pings get refused or dropped at real cadence?
  3. **The user nudge in anger**: post a user entry mid-run, nudge one
     seat, time how long the room takes to converge on it.
  4. **STOP-with-final-entry**: close a bridge and verify every seat
     learned of the close without a poll.
- The evidence standard for this spec's own claims: the native-messaging
  behavior cited here is from the official documentation as read 2026-08-23
  and one verified same-machine round-trip. Multi-seat ping behavior, hold
  behavior under mixed modes, and throttle thresholds under real cadence are
  **unobserved** -- the shakedown exists to observe them.

## 14. Open questions for the operator

1. Set `crossSessionInbound: accept` fleet-wide on both machines? Removes F3's
   mixed-mode holds entirely -- appropriate only on a fleet where every
   session is the same person's; the trade is that any of your sessions can
   always message any other without a hold gate.
2. Does the home machine's Claude Code meet the native-Windows version floor
   (v2.1.234+)? If not, home bridges stay watch-mode until it does.
