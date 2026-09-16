# Two-agent peer review protocol

Two coding agents review the same frozen diff. Each is assigned a fixed letter, `A` or `B`,
by the human. They alternate turns, communicating only through files in a mailbox directory.
The human launches A and B once with explicit roles and branch. After kickoff, both agents
monitor autonomously, take their turns without another prompt, and stop only when the session
closes, blocks, or the human cancels.

This file is the complete specification. It assumes nothing beyond a POSIX shell, `git`, and
the ability to read and write files. It is not specific to any agent app.

## Required inputs

Two things must be supplied by the human, and neither has a default:

- **Role** — `A` or `B`. Never infer it from the state file or from which agent moved last.
- **Branch** — the branch under review, named explicitly. Do not fall back to whatever the
  checkout happens to have checked out.

Refuse to start if either is missing, and refuse outright if the named branch is the main branch
(`master` or `main`). Reviewing the trunk against itself is an empty diff and always means the
branch argument was wrong or omitted.

The branch is a ref, not a checkout. The diff is read with `git diff BASE..<branch>`, so nothing
needs to be checked out and neither agent disturbs the other's working tree.

## Ground rules

1. **Review only.** Never modify a file under review. Never stage, commit, push, or amend.
   Never comment on the pull request or write to GitHub in any way. Fixes are *proposed* as
   diffs inside your round file; the human applies what they accept.
2. **Write only inside the mailbox.** The only files you create or change are the mailbox
   files listed below.
3. **Write only on your turn, and only while holding the lock.** Reading needs no lock.
4. **The diff is frozen** at `BASE..HEAD` recorded in `scope.md`, where `HEAD` is the branch tip
   pinned at init. Check `git rev-parse <branch>` against it before reviewing. If the branch has
   moved, stop and tell the human — never silently review a different diff.
5. **Five rounds maximum** (ten round files). Round 5 is the last; the session force-closes after it.
6. **New findings in rounds 1–3 only.** Rounds 4 and 5 are for resolving what is already on the table.
7. **A blocker is raised, not worked around.** See **Blockers and the hold**.
8. **Monitoring is autonomous and read-only.** Never acquire the lock or write while waiting.
9. **Blind means no-peek.** Neither agent reads peer findings or existing PR review comments
   before both independent finding sets are fixed and B's commitment is verified.
10. **Claims are immutable.** Corrections are appended as acknowledged amendments; original
    claims and evidence are never silently rewritten.

## Mailbox layout

```
<mailbox>/
  PROTOCOL.md       this file, pinned for the session
  scope.md          neutral scope; no existing review findings
  diff.patch        exact frozen BASE..HEAD input
  state.json        phase, round, turn and status
  blind/
    B.commit.json   digest/size commitment; no finding content
    A.md            A's fixed independent findings
    B.md            B's revealed independent findings
  external-comments.md  fetched only after blind reveal, when applicable
  findings.md       live ledger: immutable IDs and current state
  blocker.md        present only while a blocker is held
  lock/             directory; its existence means the lock is held
    owner.txt       "<letter> <ISO-8601 timestamp>", plus " hold" on a blocker hold
  rounds/
    01-A.md
    01-B.md
    02-A.md
    ...
  summary.md        written only by A during finalization
```

## Init (once per review, by agent A)

Nobody types a ticket or a session id. Those come from the branch, which the human named:

- **Branch slug** — the given branch name with `/` replaced by `-`. The ticket, when there is
  one, is already in it. Fetch first (`git fetch`) so the ref is current.
- **Session** — a new directory named `<YYYYMMDD-HHMMSS>`. Several reviews of the same branch
  can be open at once, so the session directory, not the branch, is what identifies a review.
- **Mailbox** — `<repo-root>/research/peer-review/<branch-slug>/<session>/`
- **BASE** — `git merge-base <branch> origin/<main-branch>`, unless the human names another base.
- **HEAD** — `git rev-parse <branch>`, pinned as a full SHA.

Write `scope.md` with: branch, session id, full `BASE` and `HEAD` SHAs, PR number if there is
one, the diff file list (`git diff --stat BASE..HEAD`), the absolute mailbox path, the absolute
checkout path both agents must use, the round cap (5), and the object digest and byte length of
`diff.patch`. Write the exact `git diff --binary BASE..HEAD` bytes to `diff.patch`; both agents
review that file and the repository at the pinned refs. Use `git hash-object diff.patch` for the
digest and `wc -c < diff.patch` for the byte length so both agents can reproduce the checks with
the protocol's required tools.

Do not fetch, summarize, or copy existing bot/human review comments during init. They are
quarantined until blind reveal.

Write `state.json` as
`{"phase":"blind_b_commit","round":0,"turn":"B","status":"open"}`, create `blind/` and
`rounds/`, copy this file into the mailbox, and initialize `findings.md` with an empty table.
A then releases the lock and monitors. B joins the initialized session once; after that both
agents advance autonomously.

Neither agent declares what the other one is. You are told your own letter and need nothing else
about the other side. Record your own app name in the header of each round file you write; that
is all the provenance this protocol uses.

Both agents must use the **same absolute mailbox path**. The handoff line carries it. If the two
agents run in different checkouts, that shared path wins over whatever either checkout would
resolve on its own.

## Finding the session

Given an explicit mailbox path, use it. Otherwise look under
`<repo-root>/research/peer-review/<branch-slug>/`:

- exactly one session with `status` open → that is the one
- several open → list them with session id, round and turn, and ask the human which
- none open → there is nothing to continue

## Taking a turn — exact sequence

1. Read `state.json`.
   - `status` is `closed` → follow **Chat report**, stop.
   - `status` is `blocked` → follow **Blockers and the hold**, do not monitor past it.
   - `turn` is not your letter → follow **Monitoring**. Do not acquire the lock.
2. Acquire the lock: `mkdir <mailbox>/lock`. This is atomic, so two agents cannot both succeed.
   - If it fails, the lock is held. Read `lock/owner.txt`. If it says `hold`, it is a blocker
     hold: never break it at any age, and report the blocker to the human instead. Otherwise,
     if its timestamp is older than 30 minutes, you may break it with `rm -rf <mailbox>/lock`
     and retry — and you must record the break in your round file. Otherwise stop and report
     who holds it.
3. Write `lock/owner.txt` as `<your letter> <ISO-8601 timestamp>`.
4. Re-read `state.json`. If `turn` is no longer yours, release the lock and stop. (This closes
   the gap between checking and acting.)
5. Check `git rev-parse <branch>` against the `HEAD` in `scope.md`. If it moved, release the lock
   and report to the human. Verify `diff.patch` against the digest and byte length in `scope.md`;
   a mismatch is a blocker.
6. Do the work for the current phase (below).
7. Write the phase artifact or `rounds/NN-<letter>.md` when required.
8. Update `findings.md` during reveal, debate, or finalization.
9. Update `state.json` using **Phase transitions**.
10. Only A in `finalize_a` may write `summary.md` and set `status` to `closed`.
11. Release the lock: `rm -rf <mailbox>/lock`.
12. Report in chat, print the handoff line while the session remains open, then resume
    **Monitoring** without waiting for another human prompt.

Releasing the lock is mandatory even when you stop early or hit an error. If you cannot finish a
round, delete the lock and say so. The single exception is a blocker hold, below.

## Monitoring

Monitoring is enabled by default after each agent's one-time kickoff. It changes scheduling, not
review semantics:

1. Read `state.json` without acquiring the lock.
2. If `status` is `closed` or `blocked`, report it and stop monitoring.
3. If `turn` is the other agent, wait before checking again. Use the app's native wait mechanism;
   if none exists, use a POSIX wait. Do not write, hold a lock, or inspect the other agent's
   in-progress round while waiting.
4. When `turn` names your role, restart **Taking a turn** at step 1.
5. After your handoff, resume monitoring until the session closes, blocks, or the human cancels.

An app that cannot remain alive or wait autonomously must disclose that limitation at kickoff.
It must not pretend the other agent needs to be prompted by protocol; the limitation belongs to
that runtime.

## Phase transitions

`state.json` has this shape:

```
{"phase":"blind_b_commit"|"blind_a_publish"|"blind_b_reveal"|"debate"|"finalize_a",
 "round":0|1|2|3|4|5, "turn":"A"|"B", "status":"open"|"blocked"|"closed"}
```

### 1. `blind_b_commit` — B

B reviews `diff.patch` and the pinned repository without reading any peer or external review.
B completes the exact text it intends to reveal as `blind/B.md` in its private agent context.
Before A writes findings, B writes `blind/B.commit.json` containing the content digest, byte
length, finding count, app/runtime provenance, and timestamp—but no finding content. B sets
`phase=blind_a_publish`, `turn=A`, releases the lock, and monitors.

Compute the draft digest with `git hash-object --stdin` over the exact bytes retained for reveal;
compute the byte length over those same bytes. At reveal, `git hash-object blind/B.md` and
`wc -c < blind/B.md` must match the commitment.

If B loses the committed draft before reveal, it cannot recreate or revise it after A publishes.
Set a blocker and restart the blind phase or abandon the session.

### 2. `blind_a_publish` — A

A reviews the same frozen input without seeking B's private draft and writes `blind/A.md`.
A sets `phase=blind_b_reveal`, `turn=B`, releases the lock, and monitors.

### 3. `blind_b_reveal` — B

Before reading `blind/A.md`, B writes the exact committed draft to `blind/B.md` and verifies its
digest and byte length against `blind/B.commit.json`. A mismatch is a blocker, never a warning.
Only after verification may B read A's blind file.

B then fetches existing PR review comments, if relevant, into `external-comments.md`; they enter
the ledger as `X1`, `X2`, ... and carry no truth status until both agents verify them. B
initializes `findings.md` from the fixed blind files using origin-qualified IDs (`A1`, `A2`, ...,
`B1`, `B2`, ...; overengineering items use `OA1`, `OB1`, ...), sets `phase=debate`,
`round=1`, `turn=A`, releases the lock, and monitors.

### 4. `debate` — A then B

A writes `rounds/NN-A.md`, then sets `turn=B` at the same round. B writes `rounds/NN-B.md`, then
increments the round and sets `turn=A`.

If the close test passes after A's turn, A may proceed directly to `finalize_a`. If it passes
after B's turn, B sets `phase=finalize_a`, `turn=A`. B finishing round 5 always routes to
`finalize_a`, whether converged or diverged. B never closes or presents the session.

### 5. `finalize_a` — A

A verifies ledger completeness, claim fidelity, amendments, and the close result. A writes
`summary.md`, appends the round history, sets `status=closed`, releases the lock, and presents the
absolute summary link to the human. A is the sole finalizer.

## Blockers and the hold

A blocker is anything that stops you from reviewing to the depth this protocol demands — that
is, anything that prevents you from establishing reachability and a concrete failure scenario
rather than guessing. Typical cases: the code will not build or import, tests cannot run, a
fixture or credential is missing, the diff depends on an unmerged change, a referenced file is
absent, the base looks wrong, or the claim can only be settled against data you have no access
to and must not obtain yourself.

Both blind reviewers carry this duty. B reviews first under the no-peek gate, so B must hold on
any defective basis it discovers; A independently does the same during its blind phase. Either
agent may use the mechanism during debate.

On hitting one:

1. Write `blocker.md`: current phase/round/role, what is blocked, the specific thing needed to
   unblock it, and what you were and were not able to verify without it.
2. Do **not** flip `turn`. Leave it on your own letter and set `status` to `blocked`.
3. **Keep the lock.** Rewrite `lock/owner.txt` as `<your letter> <ISO-8601 timestamp> hold`.
   A hold is never stale-breakable, so the other agent cannot start on a basis you already know
   is defective.
4. Report the blocker in chat and stop. Do not print a handoff line — there is no handoff.

Nothing moves until the human confirms. Then:

- **Unblocked** — verify the unblocking action actually landed, finish the round properly on the
  same turn and round number, flip `turn`, set `status` back to `open`, release the lock.
- **Told to proceed anyway** — record in `scope.md` what stayed unverifiable, and mark every
  affected area as not reviewed in depth so it carries into `summary.md`. Findings that needed
  the missing evidence cap at P3 or become `needs-evidence`. Then continue as above.
- **Abandoned** — set `phase=finalize_a`, `turn=A`, keep `status=open`, and release the lock.
  A records the blocker as the reason in `summary.md` and closes the session.

Do not silently narrow the scope to whatever happens to be reviewable. An unflagged blocker
turns the whole session into two agents agreeing about code neither of them could actually check.

## The work

### Blind phase

Both agents inspect the full frozen scope independently. Blind files contain:

- defect findings using the format below
- an `## Overengineering candidates` section using the separate format below, or an explicit
  `None found` with the areas checked
- tests, queries, and source material actually inspected
- the agent's app, model family when known, runtime, and context provenance

The blind packet contains no peer findings and no existing PR review comments. Requirements and
repository facts supplied in `scope.md` are `GIVEN`; they do not count as corroboration until an
agent independently verifies and records them as `DERIVED`.

### Debate rounds 1–5

Every debate turn contains:

- an adversarial verdict on **every** open finding and overengineering candidate raised by the
  other side; no silent skips
- an adversarial verdict on every external comment still open
- a defense, acknowledged amendment, or withdrawal for each own finding that was challenged
- new findings only while `round <= 3`, labeled `derived-during-debate`
- a short `Round delta` listing new IDs, terminal IDs, amendments, blockers, and evidence added

The finding set freezes after round 3. Rounds 4 and 5 only resolve existing IDs.

## Finding format

```
### <A|B><N> · P<1|2|3> · <short title>
Location: <path>:<line>
Claim: one sentence stating the defect.
Reachability: what calls this, under which config / params / data / environment. If you
  cannot show it is reachable, mark it P3 or withdraw it.
Failure scenario: concrete inputs or state, leading to the wrong output or crash.
Evidence: what you actually read or ran — file:line, or command and its output.
Evidence provenance: GIVEN <source> | DERIVED <agent, source locator>
Falsifier: the bounded observation that would disprove the claim.
Proposed patch: minimal diff. Optional.
```

Priorities: **P1** a correctness bug shown to be reachable. **P2** real but narrow, conditional,
or a risk rather than a live defect. **P3** quality, simplification, or a nit.

Claims and falsifiers are immutable. A later correction is an `Amendment` that quotes the old and
new text and becomes effective only when the originator explicitly accepts it.

## Overengineering format

Every blind review judges whether changed implementation items are unrelated to the stated goal
or impose large complexity, operational cost, or review surface for negligible gain:

```
### O<A|B><N> · <short title>
Location: <path>:<line>
Core objective: the requirement this change is meant to serve.
Excess: what is unrelated or disproportionate.
Cost: implementation, maintenance, operational, or review burden.
Expected gain: concrete benefit and its likely size.
Minimal alternative: smaller implementation that preserves the core objective.
Evidence provenance: GIVEN <source> | DERIVED <agent, source locator>
Falsifier: what would show the complexity is necessary and proportionate.
```

Do not call code overengineered merely because it is large or unfamiliar. The candidate must
connect cost to a small, unproven, or out-of-scope gain. Confirmed overengineering is a
recommendation to reject or trim that implementation item; it is never edited automatically.

## Verdicts

Counter-review begins from attempted disconfirmation. For every peer or external finding, record:

```
Attack attempted: the concrete guard, counterexample, test, or intended-behavior argument tried.
Independent evidence: a DERIVED source locator or command result.
Verdict: <one exact verdict below>
```

Use exactly one defect verdict:

- `confirmed` — you independently re-derived it. Include your own reachability check and
  failure scenario after a concrete falsification attempt failed.
- `rejected: <reason-class>` — one of `unreachable`, `guarded-upstream`, `test-disproves`,
  `misread-code`, `intended-behavior`, `out-of-scope`, `duplicate-of-<ID>`. Include the
  evidence that kills it.
- `needs-evidence: <what exactly>` — you can neither confirm nor kill it. Name the specific
  artifact that would settle it.

Use exactly one overengineering verdict:

- `confirmed-overengineering` — the peer independently verified disproportionate cost and the
  smaller alternative.
- `rejected: related-and-proportionate` — evidence shows the implementation is needed or its
  cost is proportionate to the expected gain.
- `needs-evidence: <what exactly>` — the cost, gain, or necessity cannot yet be established.

On your own findings, `withdrawn` means you accept the rebuttal.

## Anti-collusion rules

Two agents reviewing together drift toward agreement. That is the main failure mode of this
protocol, and these rules exist to resist it.

- Never write `confirmed` without your own verification trail.
- Every counter-review starts with an attack. Lenient agreement, praise, and restatement are not
  verdict work.
- While any peer finding is open, try to kill that specific claim using its falsifier before
  looking for supporting evidence.
- Mechanism-correct is not the same as reachable. A finding whose path nothing can reach is
  `rejected: unreachable` however elegant the mechanism.
- A round with zero rejections and zero `needs-evidence` is a signal you did not really try.
  If that happens, list the concrete attacks attempted and why each failed.
- Priority inflation is itself a defect. Challenge any P1 that carries no reachability argument.
- Do not negotiate toward the middle. A finding is either shown reachable or it is not; there
  is no splitting the difference to end a round.
- Repetition of `GIVEN` evidence does not corroborate it. Only independently `DERIVED` evidence
  can support confirmation.
- Treat overengineering claims adversarially too: require evidence that the gain is genuinely
  small and the alternative genuinely preserves the objective.

## Close test

The session is eligible for A finalization when all hold:

- every defect finding is terminal: peer-confirmed; peer-rejected and origin-withdrawn; or an
  accepted duplicate. A defended rejection or `needs-evidence` remains open
- every overengineering item is terminal: peer `confirmed-overengineering`; or peer
  `rejected: related-and-proportionate` and origin-withdrawn. A defended rejection or
  `needs-evidence` remains open
- every external `X<N>` item has matching terminal verdicts from A and B; disagreement remains
  open
- neither side raised a new finding in its most recent round.

Eligibility never closes the session directly. It routes through `finalize_a`. If the conditions
do not hold when B finishes round 5, B still routes to `finalize_a`; A records **diverged at
round 5** and preserves each open position.

## findings.md

A single table, current state only. Detail stays in the immutable blind files and append-only
round responses.

```
| ID | Kind | P | Title | Location | Raised by | Peer verdict | Origin response |
|----|------|---|-------|----------|-----------|--------------|-----------------|
| A1 | defect | 1 | ... | path:line | A | confirmed | stands |
| OB1 | overengineering | 3 | ... | path:line | B | confirmed-overengineering | stands |
```

## summary.md

Written only by A in `finalize_a`. A derives status from `findings.md` and the round artifacts;
summary prose may explain ledger state but may not assign or alter it. Sections, in order:

1. **Status** — `converged at round N` or `diverged at round 5`.
2. **Confirmed findings** — ordered P1 → P3, each with location, the failure scenario, and the
   proposed patch if there is one.
3. **Rejected items**
   - **Overengineering rejected from the implementation** — every
     `confirmed-overengineering` item with cost, expected gain, and minimal alternative.
   - **Review findings rejected as false positives** — one line each: finding, reason class,
     who killed it.
4. **Open / diverged** — each item with A's position and B's position, and what evidence would
   settle it.
5. **Round history** — appended last: one brief entry for the blind phase and each debate round,
   naming each agent's new findings, verdict changes, amendments/withdrawals, blockers, and
   decisive evidence. Keep each agent-turn to one or two lines.

## Chat report

Agent A alone presents the final result. After writing `summary.md` and closing state, A says:

```
Peer review <branch-slug>/<session>: <converged at round N|diverged at round 5>.
[Open the final peer-review summary](<absolute mailbox path>/summary.md)
```

If B observes `status=closed`, B reports only that A owns final presentation and stops. B never
links or restates the summary.

## Handoff line

End every non-final completed turn with exactly this. It is an observability aid; autonomous
monitoring, not human pasting, advances the session:

```
Peer review <branch-slug>/<session>: phase <phase>, round <N> written by <letter>. Next: <other letter> — mailbox <absolute mailbox path>
```
