---
name: deepseek-adversarial-review
description: "Token-cheap adversarial PR/MR review loop for DeepSeek models. One fresh read-only reviewer sub-agent per round, briefed to stay near 40-60 turns and ~120k context, with the coordinator applying fixes and running the checks. Same two-immediately-consecutive-clean-rounds stop condition as adversarial-mr-review, at a fraction of the tokens. Use when running a review-fix-repeat loop on a cheap high-context model, or when an adversarial review is burning too many tokens."
---

# DeepSeek Adversarial Review

Same review, a fifth of the tokens. Independence and the two-clean-rounds stop condition stay exactly as they are. What changes is the brief: a round reads what the change touches instead of re-surveying the repository, and the suites stay out of the reviewer.

## Why rounds get expensive

A turn re-sends the whole conversation, so cost is roughly `turns × context`. Measured on real runs of the sibling skill:

| Run | Turns | Cache read | Avg context |
|---|---|---|---|
| Onboarding PR, 26 rounds | 42–76 | 5.0–12.9M | 119k–170k |
| Push-after-sign-in PR | 32–50 | 2.0–4.6M | 61k–97k |
| Money PR, 3 rounds | 106–195 | 19–48M | 179k–245k |

Where the expensive round's 47.7M went: whole-file reads 42%, reasoning over 195 turns 27%, the same `git diff` re-fetched 18 times 17%, greps 9%, test suites 4%.

The 42–76-turn rounds were the cheap ones, and they got there by not opening whole files, not re-fetching the diff, and not carrying a checklist of things to verify.

## Rules that keep a round cheap

1. **One diff fetch per revision.** `git diff <base>...<rev> > /tmp/review.diff`, then slice it with `sed -n 'a,bp'`. Never run `git diff base...head -- <paths>` again for each area; that was 6.4M on its own.
2. **Ranges, not files.** `sed -n '120,180p' file` or a targeted read. Fifty whole-file reads were 16M; the cheap rounds made 2–15 reads.
3. **Batch the searches.** One `rg -e a -e b -e c` beats twelve calls, and every call is re-billed per turn after it.
4. **No builds, no suites in the reviewer.** The coordinator runs them once, redirecting to a log and reading back a summary line: `... > /tmp/t.log 2>&1; tail -5 /tmp/t.log`. A reviewer may run one targeted test file when it proves a specific finding.
5. **Scope, never a checklist.** One sentence naming the subsystem and the behaviour. A list of twenty functions to verify is an instruction to spend 150 turns verifying them — that was the whole regression.
6. **Stop at about 60 tool calls.** If the evidence is not there yet, report what was found and say what was not examined. A 195-turn round costs 9× a 40-turn one and finds the same class of issue.

## The brief

Copy this shape, fill the four bracketed fields, change nothing else:

```
You are a subagent spawned by another session. You are an independent,
adversarial, architect-level code reviewer. Conduct exactly ONE review round.
You are READ-ONLY: do not edit, create, commit, or push anything.

Target
- Repository: [path]
- Revision under review: [sha]
- Base: [sha] (origin/main)
- Scope: [one sentence — the subsystem and the behaviour changed]

Governing rules: read [AGENTS.md / CLAUDE.md paths for this area]. Also check
docs/accepted-findings/; anything recorded there is decided and must not be
reported unless the code changed.

Getting the diff
  cd [path]
  git diff [base]...[rev] > /tmp/review.diff
  git diff --name-status [base]...[rev]
Read /tmp/review.diff in slices (sed -n). Do not re-run git diff per file.

How to work
- Read the changed functions and the call sites of changed symbols. Read a
  function's line range, not the whole file.
- Batch symbol searches: rg -e a -e b -e c apps supabase packages scripts.
- Do not run builds or test suites. If a test is needed to prove a finding,
  run that one file, redirect it to a log, and read back the last few lines.
- Stop at about 60 tool calls.

What counts as a finding
- Correctness: null/empty, error paths, silently lost failures.
- Regressions: existing callers, public contracts, previously working behaviour.
- Architecture: misplaced logic, new coupling, duplicate paths that will diverge.
- Concurrency and money invariants when the change touches them.
- Missing coverage for changed behaviour.
Every finding names a mechanism: a state or input and the wrong result, or the
defect the change will produce later and how. Architecture and scope findings are
findings; "consider extracting" with no mechanism behind it is not.
Do not report purely speculative refactors of code this change does not touch.

Output
- severity, file:line, mechanism, what breaks, why the tests do not catch it.
- One line each, most severe first, at most eight.
- Then the revision reviewed and the checks actually run (command + result).
- If nothing: "ZERO FINDINGS" and list what was verified.
```

## The loop

1. Spawn a reviewer with the brief above, as a new agent with none of this conversation. Never reuse one, never brief it with earlier findings or the streak.
2. The coordinator fixes every finding, in one pass per file, and runs the checks the fixes need.
3. Write the round to `<git-dir>/adversarial-review/round-<n>.md` and one line to `ledger.md`. Keep the context flat; do not paste reports into it.
4. Spawn a different fresh reviewer for the next round, on the full current diff — not on the delta.

A round is clean only when its reviewer reports zero findings against the revision it was given.

## Stop condition

Unchanged from `adversarial-mr-review`, because it is the accuracy guarantee and not a cost lever:

- Two **immediately adjacent** rounds, N and N+1, both clean, from **two different fresh reviewers**, on the **identical unchanged diff**.
- Every attempt counts. An attempt that finds something, is incomplete, interrupted, blocked, or times out resets the streak to 0, even if no source changed.
- Any source or test change resets the streak to 0.
- Freeze the tree between the two certifying rounds.
- Report the two attempt numbers, the distinct reviewers, and the reviewed revision.
- After roughly 8–10 rounds without convergence, report the unresolved findings and the blocker instead of claiming completion.

## What not to do

- Do not shrink the diff the reviewer sees, and do not shard it across reviewers: two clean rounds must both see the same complete diff, and a change in one slice with its caller in another belongs to neither.
- Do not let the reviewer run the suite to "check nothing broke". That is the coordinator's job and it is 4% of the tokens but most of the wall clock and a large share of the turns.
- Do not add "also verify each of these N functions" to the brief. It is the single largest cost multiplier measured on this project.
