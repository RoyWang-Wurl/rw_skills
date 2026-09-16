---
name: agentic-peer-review
description: Run one turn of a turn-based peer review in which two different coding agents (for example Claude Code and Codex) review the same frozen diff or PR, alternating to propose findings, challenge each other's findings, and converge. Communication is through files in a gitignored mailbox, never chat-to-chat. Use when asked to start, continue, monitor, or check the status of an agent peer review.
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

**No open session, or `--new`** → init per `PROTOCOL.md`: `git fetch`, then pin `BASE` and `HEAD`
off the named branch. For a PR target, pin the SHAs with
`gh pr view <n> --json baseRefName,headRefOid,number`. Take round 1 only if you are A.

**One open session** → take your turn, following the turn sequence exactly. Stop if the turn is
not yours unless explicit monitoring is active.

**Several open sessions** → list them with session id, round and turn, and ask which one.

**`status` is `blocked`** → the holder is waiting on the human. If you are the holder, check
whether the unblocking action landed and resume that same round; if you are not, report the
blocker and stop. Never break a `hold` lock.

**Session closed** → read `summary.md`, give the chat report, write nothing.

## Blockers

If anything stops you reviewing to the depth the protocol requires — the code will not import,
tests will not run, a fixture or credential is missing, the claim can only be settled against
data you must not touch — do not quietly review whatever is left. Follow **Blockers and the
hold** in `PROTOCOL.md`: flag it in the round file, keep the lock as a `hold`, set `status` to
`blocked`, report in chat, and wait for the human. This matters most for A in round 1, since a
defective basis costs both sides every remaining round.

## Doing the review pass

For your own independent findings, review the frozen `BASE..HEAD` range against the actual code
and tests, and treat your notes as raw material, not as the round file. Then apply this skill's
own bar before anything reaches your round file:

- every finding needs the reachability argument and the concrete failure scenario that
  `PROTOCOL.md` requires; drop or demote whatever cannot carry one
- verify each premise against the actual code and, where the claim depends on data or config,
  against live data before asserting it
- as agent B in round 1, do this pass and write your own findings *before* reading A's file

## Verdicts on the other side's findings

Re-derive each one yourself. Read the cited lines, trace the callers, and try to construct the
failing input. Mark `confirmed` only when you got there independently; `rejected: <reason-class>`
with the evidence that kills it; `needs-evidence: <what>` when neither holds. Restating the other
agent's argument back to it is not verification, and a round in which you rejected nothing needs
an explicit note on what you tried to break.

## Finishing a turn

Write the round file, update `findings.md`, update `state.json`, release the lock, then print
the handoff line from `PROTOCOL.md` so the human can paste it into the other app. Release the
lock even if you abort mid-turn.

Stop after handoff by default. If the human explicitly asks you to monitor, use the app's wait
mechanism to watch `state.json` read-only; do not acquire the lock or write while it is the other
agent's turn. Take the next turn when `state.json` names your role, then resume monitoring after
handoff until the session closes, blocks, or the human cancels. If the app has no native wait
mechanism, use a POSIX wait between read-only state checks.
