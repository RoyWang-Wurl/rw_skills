# Two-agent peer review protocol

Two coding agents review the same frozen diff. Each is assigned a fixed letter, `A` or `B`,
by the human. They alternate turns, communicating only through files in a mailbox directory.
A human triggers each turn or explicitly asks an agent to monitor for its next turn.

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
8. **Monitoring is read-only.** Never acquire the lock or write while waiting for the other agent.

## Mailbox layout

```
<mailbox>/
  PROTOCOL.md       this file, pinned for the session
  scope.md          what is under review; written once at init
  state.json        {"round": 1, "turn": "A", "status": "open"|"blocked"|"closed"}
  findings.md       live ledger: one row per finding, current status
  lock/             directory; its existence means the lock is held
    owner.txt       "<letter> <ISO-8601 timestamp>", plus " hold" on a blocker hold
  rounds/
    01-A.md
    01-B.md
    02-A.md
    ...
  summary.md        written by whoever closes the session
```

## Init (once per review, by either agent)

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
checkout path both agents must use, and the round cap (5).

Write `state.json` as `{"round": 1, "turn": "A", "status": "open"}`, create `rounds/`, copy this
file into the mailbox, and initialize `findings.md` with an empty table.

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
   - `status` is `closed` → report the summary in chat (see **Chat report**), stop.
   - `status` is `blocked` → follow **Blockers and the hold**, do not monitor past it.
   - `turn` is not your letter → if explicit monitoring is active, follow **Monitoring**;
     otherwise say whose turn it is and stop. Do not acquire the lock.
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
   and report to the human.
6. Do the work for this round (below).
7. Write `rounds/NN-<letter>.md`, zero-padded round number.
8. Update `findings.md`.
9. Update `state.json`: flip `turn` to the other letter; if you are B, increment `round`.
   If the close test passes, or you just finished round 5 as B, set `status` to `closed`.
10. If closing, write `summary.md`.
11. Release the lock: `rm -rf <mailbox>/lock`.
12. Report in chat, then print the handoff line. If explicit monitoring is active and the
    session remains open, resume **Monitoring**.

Releasing the lock is mandatory even when you stop early or hit an error. If you cannot finish a
round, delete the lock and say so. The single exception is a blocker hold, below.

## Monitoring

Monitoring is enabled only when the human explicitly requests it. It changes scheduling, not
review semantics:

1. Read `state.json` without acquiring the lock.
2. If `status` is `closed` or `blocked`, report it and stop monitoring.
3. If `turn` is the other agent, wait before checking again. Use the app's native wait mechanism;
   if none exists, use a POSIX wait. Do not write, hold a lock, or inspect the other agent's
   in-progress round while waiting.
4. When `turn` names your role, restart **Taking a turn** at step 1.
5. After your handoff, resume monitoring until the session closes, blocks, or the human cancels.

## Blockers and the hold

A blocker is anything that stops you from reviewing to the depth this protocol demands — that
is, anything that prevents you from establishing reachability and a concrete failure scenario
rather than guessing. Typical cases: the code will not build or import, tests cannot run, a
fixture or credential is missing, the diff depends on an unmerged change, a referenced file is
absent, the base looks wrong, or the claim can only be settled against data you have no access
to and must not obtain yourself.

Agent A carries this duty in round 1, because A reviews first and a defective basis wastes both
sides' rounds. B may use the same mechanism later if it hits a genuine blocker.

On hitting one:

1. Write your round file with a `## Blocker` section: what is blocked, the specific thing needed
   to unblock it, and what you were and were not able to verify without it.
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
- **Abandoned** — set `status` to `closed`, note the blocker as the reason in `summary.md`, and
  release the lock.

Do not silently narrow the scope to whatever happens to be reviewable. An unflagged blocker
turns the whole session into two agents agreeing about code neither of them could actually check.

## The work

**Round 1, agent A** — independent review of the frozen diff. Raise findings `F1`, `F2`, …

**Round 1, agent B** — in this order:
1. Review the diff yourself **before reading A's round file**, and write your own findings.
2. Then read `01-A.md` and issue a verdict on each of A's findings.

Doing it in that order matters. Reading A's list first anchors you to A's framing and is the
fastest way to turn a peer review into an echo.

**Rounds 2–5, either agent** — in your round file:
- a verdict on **every** open finding raised by the other side; no silent skips
- defend or withdraw each of your own findings that was challenged
- new findings only while `round <= 3`

## Finding format

```
### F<N> · P<1|2|3> · <short title>
Location: <path>:<line>
Claim: one sentence stating the defect.
Reachability: what calls this, under which config / params / data / environment. If you
  cannot show it is reachable, mark it P3 or withdraw it.
Failure scenario: concrete inputs or state, leading to the wrong output or crash.
Evidence: what you actually read or ran — file:line, or command and its output.
Proposed patch: minimal diff. Optional.
```

Priorities: **P1** a correctness bug shown to be reachable. **P2** real but narrow, conditional,
or a risk rather than a live defect. **P3** quality, simplification, or a nit.

## Verdicts

Responding to the other side's finding, use exactly one:

- `confirmed` — you independently re-derived it. Include your own reachability check and
  failure scenario. Restating the other agent's reasoning is not verification.
- `rejected: <reason-class>` — one of `unreachable`, `guarded-upstream`, `test-disproves`,
  `misread-code`, `intended-behavior`, `out-of-scope`, `duplicate-of-F<N>`. Include the
  evidence that kills it.
- `needs-evidence: <what exactly>` — you can neither confirm nor kill it. Name the specific
  artifact that would settle it.

On your own findings, `withdrawn` means you accept the rebuttal.

## Anti-collusion rules

Two agents reviewing together drift toward agreement. That is the main failure mode of this
protocol, and these rules exist to resist it.

- Never write `confirmed` without your own verification trail.
- While any of the other side's findings are open, make at least one genuine rebuttal attempt
  per round. Go looking for the reason a finding is wrong, not only reasons it is right.
- Mechanism-correct is not the same as reachable. A finding whose path nothing can reach is
  `rejected: unreachable` however elegant the mechanism.
- A round with zero rejections and zero `needs-evidence` is a signal you did not really try.
  If that happens, state explicitly what you attempted to break and failed to break.
- Priority inflation is itself a defect. Challenge any P1 that carries no reachability argument.
- Do not negotiate toward the middle. A finding is either shown reachable or it is not; there
  is no splitting the difference to end a round.

## Close test

The session is closed when both hold:

- every finding has a terminal verdict from both sides — `confirmed`, `rejected`, `withdrawn`,
  `out-of-scope`, or `duplicate`, and
- neither side raised a new finding in its most recent round.

Otherwise the session force-closes once B finishes round 5. Anything not terminal at that point
is recorded as **open / diverged**, carrying both sides' positions.

## findings.md

A single table, current state only. Detail stays in the round files.

```
| ID | P | Title | Location | Raised by | Status |
|----|---|-------|----------|-----------|--------|
| F1 | 1 | ...   | path:line| A         | confirmed |
```

## summary.md

Written by whoever closes. Sections, in order:

1. **Status** — `converged at round N` or `diverged at round 5`.
2. **Confirmed findings** — ordered P1 → P3, each with location, the failure scenario, and the
   proposed patch if there is one.
3. **Rejected** — one line each: finding, reason class, who killed it.
4. **Open / diverged** — each item with A's position and B's position, and what evidence would
   settle it.

## Chat report

Both agents report in their own chat when the session closes — the one that closes it, and the
other on its next run. Say:

- the convergence status,
- then the findings in priority order, P1 first, one line each,
- or, if diverged, the open items with each side's position.

Nothing else. No preamble, no restating the protocol.

## Handoff line

End a completed turn with exactly this, so the human can paste it into the other app:

```
Peer review <branch-slug>/<session>: round <N> written by <letter>. Next: <other letter> — point that agent at <absolute mailbox path>/PROTOCOL.md
```
