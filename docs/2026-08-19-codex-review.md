# Codex Review - 08/19/2026

> **DISPOSITION 2026-08-19, same day, all five recommendations acted on in
> 0.11.0** (`23515cb`, `475028a`). (1) README argument order: verified against
> the command's contract and fixed. (2) High-water reads: the scenario is
> correct and also indicted the maintainer's own `>=` merge-time fix as a
> second filter on the same broken premise; the Loop now diffs `(session, N)`
> identity pairs and prescribes no numeric threshold. (3) "Before you propose
> a rule" moved to `CONTRIBUTING.md`; the PR gate landed in the PR template.
> (4) Close-out ownership: liveness clauses removed in all three places;
> first-to-observe-with-no-close-out-on-file owns it, and reaching the user is
> deliberately ownerless since the user watches the file. (5) Volatile counts
> removed from the README rather than refreshed. Two amendments to the
> review's remedies: the identity-pair diff is still model-executed
> bookkeeping, the layer behind every shipped defect, so the scheduled 1.0
> script proposal (`2026-08-19-review-what-this-is-becoming.md`) remains the
> real fix; and finding 5 described a README replaced an hour before the
> review ran, but its principle -- do not duplicate volatile counts -- was
> adopted over that refresh. The do-not-build list matches the 1.0 review's
> independently; where the two disagree (script now vs script on next
> failure), the sequencing is the review's and the script stays scheduled.

The core idea is sound, and I would preserve its intentionally small
architecture: one shared file, peer sessions, no service, no coordinator. The
bridge is valuable precisely when independently informed sessions need to
challenge each other.

The immediate problems are correctness and instruction load, not a need for
more features.

## Fix first

### 1. The README reverses the command arguments

The executable contract says `<session-name> [topic-or-path]` in
`commands/bridge.md`, but the README says topic first and shows:

```text
/bridge api-shape-review request-validation
/bridge api-shape-review gateway-timeouts
```

Following the command literally, those sessions create or join different files
and both call themselves `api-shape-review`. The example cannot produce the
illustrated conversation.

Pick one contract and align everything immediately. The least disruptive fix
is correcting the README examples; changing the command order is more intuitive
but breaking.

### 2. The unread-entry algorithm can still silently lose entries

The command currently combines three incompatible ideas:

- Process numbers higher than the high-water mark.
- Diff headers against handled numbers.
- Filter `N >= high-water` because numbers collide.

The recent `>=` correction catches an entry sharing the boundary number. It
does not catch a delayed append with a lower number:

```text
A reads highest=10 and prepares [11]
B appends [11]
C appends [12] and another seat handles through 12
A finally appends its [11]
```

An `>= 12` read misses A's newly appended `[11]`. The protocol itself correctly
says append order and number order diverge.

Remove the high-water concept. Track handled identities as `(session-name, N)`
and diff the complete header list against that set. That is simpler and closes
equal-number and late-lower-number cases without adding locks or new
identifiers.

## Reduce the executable prompt, not the evidence archive

The command grew from 332 to 777 lines; `protocol.md` grew from 230 to 988. The
latest trim correctly concluded that most remaining command growth is rules,
not duplicated storytelling.

I would not split `docs/protocol.md` yet. It is an evidence archive and is not
loaded during normal bridge use. Give it a generated table of contents if
navigation becomes annoying, but let the historical evidence remain coherent.

The runtime command is the costly file. One easy removal is "Before you propose
a rule": those 33 lines govern protocol maintainers, not bridge participants.
Move them to `CONTRIBUTING.md` or `CLAUDE.md`. A session using `/bridge` should
not pay for instructions about amending the plugin.

Then organize the remaining command as a short lifecycle:

- Resolve and join
- On every wake
- Before every append
- DONE and close
- Cross-repo carry rules
- Safety boundaries

Each rule should get one consequence sentence; the complete incident remains in
`protocol.md`.

## Simplify ownership

The round-cap section says whoever notices the cap owns the close-out unless the
creator is visibly replying, then says the creator posts it unless the creator
is not live. But the command later correctly says liveness is not observable.

Remove the creator special case. Whoever first observes the cap and sees no
existing close-out writes it. This matches the successful "responsibility
belongs to whoever is looking" pattern already used for STOP and avoids
inventing a liveness signal.

## Repair documentation drift

The README says five runs in two places and "both changes merged so far," while
`protocol.md` says six runs and GitHub has ten merged pull requests.

Avoid duplicating volatile counts. The README can say "in regular use across two
Windows machines" and link to the protocol's current evidence statement.

## Tighten the pull-request loop without adding bureaucracy

The remote has ten merged pull requests, no open pull requests, and no open
issues. The pull-request bodies are unusually good: concrete observations,
rejected alternatives, and explicit unverified conditions. PR #5's review is
especially strong.

The weak point is that only PR #5 has a recorded formal review. PRs #9 and #10
merged without recorded discussion, and #10 needed an immediate direct
correction on `main`.

A lightweight behavioral-PR gate is enough:

- Include the smallest transcript demonstrating the failure.
- State which existing rule is replaced or generalized.
- State the net line change to `commands/bridge.md`.
- Answer: "What two states can this rule not distinguish?"
- Record one other-seat adversarial review before merge.

No branch-protection system or elaborate test framework is necessary.

## What I would deliberately not build

For now, I would avoid:

- Basic/strict protocol modes
- Chair or coordinator roles
- A server, database, lock manager, or background daemon
- More entry types
- A formal state machine
- Splitting every incident into its own document

If the identity-set read rule fails again in practice, the next justified
escalation would be one stateless PowerShell scanner that emits entry identities,
participant state, and counts. Until then, prose simplification is cheaper.

## Recommended sequence

1. Fix argument order.
2. Replace high-water tracking.
3. Move maintainer instructions out of the runtime command.
4. Clarify close-out ownership.
5. Clean the stale README.

That improves correctness and cuts cognitive load without changing what makes
the bridge useful.
