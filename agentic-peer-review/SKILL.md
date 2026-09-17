---
name: agentic-peer-review
description: Run an autonomous turn-based peer review in which two different coding agents review the same frozen diff or PR, commit independent no-peek findings, adversarially challenge each other, and converge. An event-driven coordinator or filesystem watcher resumes each agent from versioned mailbox state. Use when asked to start, continue, monitor, or check the status of an agent peer review.
disable-model-invocation: true
---

You are one of two agents in a peer review. The other is a different agent app driven by the
same human. `PROTOCOL.md` in the skill directory is the authority on the protocol; this file
only covers how to run it from here.

Copy `PROTOCOL.md` from this skill directory into the mailbox at init, then follow the mailbox
copy for the rest of the session.

## Hard prohibitions

In this mode you review and argue. You do not change anything:

- no edits to files under review, no staging, no commits, no pushes
- no PR comments, reviews, or any other GitHub write
- no writes outside the mailbox directory

A fix is a proposed diff inside your round file. The human applies what they accept.

## Invocation

Role and branch are both required, and neither is inferred:

```
/agentic-peer-review as A --branch DST-959-CGP-hyperparams-tuning-step1
```

- `--new` — start a second session even though one is open
- `--session <dir>` — target one specific session
- `--base <ref>` — override the default merge-base at init
- `/agentic-peer-review status --branch <name>` — whose turn, which round, what is still open

Ask for whichever of role and branch was not given. Never take the role from the state file, and
never substitute the currently checked-out branch for the `--branch` argument — an omitted branch
is a question for the human, not a default. Refuse if the branch is `master` or `main`.

Do not ask what the other agent is; the protocol does not use it.

## Resolve the session

The session root is `<repo-root>/research/peer-review/<branch-slug>/`, holding one timestamped
directory per review, derived as `PROTOCOL.md` specifies. That path is excluded from git through
`.git/info/exclude` — confirm with `git check-ignore -v <path>` before writing, and stop if it
is not ignored.

**No open session, or `--new`** → only A initializes per `PROTOCOL.md`: `git fetch`, then pin
`BASE` and `HEAD` off the named branch. For a PR target, pin the SHAs with
`gh pr view <n> --json baseRefName,headRefOid,number`. B waits for A's initialized session.

**One open session** → follow its phase and turn exactly. Scheduling is autonomous by default:
prefer a configured event-driven coordinator, otherwise wait read-only on mailbox-directory
filesystem events, and poll every 30 seconds only if neither is available. Take over when
`state.json` names your role and its revision is newer than the last delivery.

**Several open sessions** → list them with session id, round and turn, and ask which one.

**`status` is `blocked`** → the holder is waiting on the human. If you are the holder, check
whether the unblocking action landed and resume that same round; if you are not, report the
blocker and stop. Never break a `hold` lock.

**Session closed** → A gives the final summary link. B reports only that A owns final
presentation, then stops.

## Autonomous kickoff and no-peek

The human launches agent A, then agent B, once per session with explicit roles and branch. A
initializes the frozen review packet, but B commits its independent review before A writes any
findings:

1. B completes its blind review and publishes only its content commitment.
2. A writes `blind/A.md` without seeing B's findings.
3. B reveals `blind/B.md`; its digest must match the commitment.
4. Only then does adversarial cross-review begin.

Existing PR review comments are not blind context. Do not fetch or read them until both blind
reviews are committed and revealed.

## Blockers

If anything stops you reviewing to the depth the protocol requires — the code will not import,
tests will not run, a fixture or credential is missing, the claim can only be settled against
data you must not touch — do not quietly review whatever is left. Follow **Blockers and the
hold** in `PROTOCOL.md`: write `blocker.md`, keep the lock as a `hold`, set `status` to
`blocked`, report in chat, and wait for the human. This applies in both blind passes and every
debate round; B carries the first basis check because B reviews first.

## Doing the review pass

For your own independent findings, review the frozen `diff.patch` and pinned repository state
against the actual code and tests. Then apply this skill's own bar before anything reaches your
blind file:

- every finding needs the reachability argument and the concrete failure scenario that
  `PROTOCOL.md` requires; drop or demote whatever cannot carry one
- verify each premise against the actual code and, where the claim depends on data or config,
  against live data before asserting it
- classify evidence as `GIVEN` or independently `DERIVED`; repetition of a given fact is not
  corroboration
- include an explicit overengineering pass on the frozen diff: identify unrelated complexity
  or large effort for negligible gain, with the core objective, cost, expected gain, and
  minimal alternative. In debate, also flag a peer finding or its proposed patch when the
  recommended change is disproportionate to the claimed defect — as a new `O<letter>N`
  item, not in place of the defect verdict

## Verdicts on the other side's findings

Counter-review is adversarial, not agreeable. For every peer finding, start by trying to falsify
it: challenge reachability, inspect upstream guards, construct a counterexample, run a focused
test, or establish intended behavior. Record `Attack attempted`, `Evidence`, and `Verdict`.
`confirmed` is valid only after a concrete attack fails and the premise is independently derived;
otherwise use `rejected: <reason-class>` or `needs-evidence: <what>`. Restating the peer's argument
is not verification. Apply the same attack to overengineering candidates before agreeing that
an implementation item or a peer finding's proposed patch should be rejected. In rounds 1–3 a
new overengineering candidate may target a peer finding; still verdict the finding as a defect.

## Finishing a turn

Write the phase artifact or round file, update `findings.md` when the phase requires it, increment
the state revision and atomically replace `state.json`, release the lock, then print the handoff
line from `PROTOCOL.md` for observability. Release the lock even if you abort mid-turn.

After handoff, a configured coordinator resumes the runtime handle registered for the next role.
The coordinator is scheduling-only: it reconciles on startup/handle registration, watches the
mailbox directory, reads terminal state even under a blocker hold, and dispatches open state only
after the lock disappears. A non-hold lock older than 30 minutes triggers a recovery dispatch to
the same role, which performs and records the protocol's stale-lock break; the coordinator never
breaks it. The coordinator keeps one delivery lease per revision and considers delivery complete
only when state advances; failed or expired deliveries are retried without concurrent duplicates.
It supplies role, branch, mailbox and expected revision and never writes mailbox files. Runtime
adapters may use the Cursor SDK, Claude Agent SDK or CLI session resume, or an equivalent
explicit prompt API.

Without a coordinator, use the app's wait mechanism around a one-shot filesystem watcher on the
mailbox directory. Use observe–arm–recheck to close the setup race, reread state after every event,
rearm after unrelated events, and reconcile periodically in case an event is dropped. Watch the
directory rather than the `state.json` inode because atomic rename can replace it. A
notification-only file hook is not sufficient to resume reasoning. Poll every 30 seconds only as
a fallback. Do not acquire the lock or inspect the peer's in-progress artifacts while waiting;
continue until the session closes, blocks, or the human cancels.

Agent A alone finalizes the session. Only B performs normal debate closure into `finalize_a`,
after both agents have completed the latest round or B finishes round 5; blocker abandonment is
an explicit exception. A audits rather than reargues terminal items. If A finds a concrete
non-terminal item was closed prematurely, A records it in `finalization.md` and returns to the
next debate round before round 5; at round 5 it is recorded as diverged. Otherwise A writes
`summary.md`, including rejected overengineering items and a brief round-by-round history, closes
the session, and gives the human a Markdown link to the absolute summary path. B never presents
the final summary. A session explicitly abandoned from a blocker is summarized as
`abandoned at <phase/round>` rather than forced into converged/diverged status.
