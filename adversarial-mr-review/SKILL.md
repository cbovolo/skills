---
name: adversarial-mr-review
description: "Runs a pull/merge request through repeated adversarial, architect-level reviews using a new independent reviewer sub-agent for every round. Fix every finding immediately and continue until two immediately consecutive fresh reviewers report zero findings; incomplete attempts reset the streak. Use when the user requests an adversarial or architect-level PR/MR review, regression hunting, or a review-fix-repeat loop."
---

# Adversarial MR Review Loop

Review the actual change, fix concrete findings, and repeat until two clean rounds occur immediately in a row. Every attempted round counts in the sequence, including one that never initializes. Every round requires a different independent reviewer; re-reading the diff yourself does not count.

## Mandatory reviewer isolation

- **Every round MUST use a newly spawned independent reviewer sub-agent.** The primary agent coordinates the review and implements fixes. It must never count its own review as a round.
- **Never reuse a reviewer**, including for the second clean round. Do not resume an old reviewer or use a follow-up task to conduct another round.
- Spawn each reviewer with the **Agent tool** as a new agent (`subagent_type: "general-purpose"`, never `"fork"`, never `SendMessage` to an earlier reviewer), so it starts with none of this conversation. Run it in the foreground (`run_in_background: false`) and wait for its report; never poll its transcript or sleep-loop on it. Supply only the target repository/worktree, current revision and base, the paths of the CLAUDE.md files that govern it, review scope, and this skill's review checklist. The reviewer must read the actual current diff and callers itself.
- Do not pass previous reviewers' findings, conclusions, clean streak, implementation explanations, or claimed test outcomes to the new reviewer. Give it the current source and raw artifacts it needs, without priming its answer.
- Each reviewer conducts exactly one round and returns findings with concrete evidence, the reviewed revision, and any checks it actually ran. Keep the reviewer read-only; the primary agent applies fixes after the report.
- Record every attempted round's number, unique reviewer name/ID, reviewed revision, outcome, and clean streak afterward. Include uninitialized and incomplete attempts; never omit or renumber them to make clean reports adjacent. Two clean reports from the same reviewer, or from the implementing agent, do not satisfy the stop condition.
- If independent reviewer tools are unavailable, state that this required review cannot be completed. Do not substitute self-review or claim independent rounds occurred.

## 1. Find the MR to review

Work out what is actually being reviewed before starting:

- If the user names a PR/MR number or pastes a URL, fetch it using the matching CLI: `gh pr diff` / `gh pr view` for GitHub, or `glab mr diff` for GitLab.
- If they say "this branch" / "my changes" / give no target, find the branch's base and review `git diff <merge-base>...HEAD`, plus any scoped uncommitted changes. Use tracking information, or `main`/`master` when appropriate.
- If no actual diff can be resolved, ask for the missing target rather than reviewing nothing or guessing.

## 2. Run rounds, back to back

Run rounds sequentially. After a round's report, fixes and appropriate local checks, start the next round immediately. Close an incomplete attempt before starting its replacement. Do not wait for CI, a timer, or another confirmation within the authorized scope.

**Each round:**

1. **Spawn a fresh reviewer** under the isolation rules above. Give it the full current review scope, including earlier fixes as they appear in the current source.
2. **Have that reviewer read the diff and trace callers**, looking specifically for:
   - Correctness: edge cases, empty/null handling, error paths, and silently lost failures.
   - Regressions: existing callers, tests, public contracts, and previously working behavior.
   - Architecture: misplaced logic, new coupling, layering violations, and leaky abstractions.
   - Concurrency and shared-state issues when the change touches shared state.
   - Security issues when the change touches auth, input handling, or secrets.
   - Missing coverage for changed behavior.
3. **Receive and report the reviewer's actual findings.** Use `ReportFindings` if available; otherwise report plainly, most severe first, with file, line, and a concrete failure scenario. Say explicitly when the reviewer found zero issues. Do not call an incomplete or blocked review clean.
4. **Fix every finding now** within scope. Reproduce concrete regressions where practical and run the checks appropriate to the fix. Never fabricate results, weaken assertions, disable tests, or change expectations just to accept broken behavior. A genuine test-harness correction must be explained and retain the intended behavioral check.
5. **End that reviewer's participation and spawn a different reviewer for the next round.** Never ask the same reviewer to certify its own round again.

**Stop condition:** attempted rounds **N and N+1** must both complete with zero findings, use two different fresh reviewers, and review the identical final unchanged diff. No intervening attempt is allowed. The primary agent's checks do not replace either independent review.

Track the clean streak explicitly:

- Start every newly requested review run at **0**; do not carry clean reports forward from an earlier run.
- A completed zero-finding round against the unchanged diff increments the streak by one.
- An attempt that ends with findings or is incomplete, cancelled, interrupted, blocked, timed out, or unable to initialize resets the streak to **0**, even when no source changed. Never skip that attempt when determining adjacency. A progress wait that yields while the same reviewer continues running is not a new or failed review attempt.
- Any source or test change resets the streak to **0**.
- Stop only at **2**, and report the two adjacent attempt numbers, distinct reviewers, and identical reviewed revision. For example, **clean → incomplete → clean** ends at **1**, not 2; another fresh clean round is required.

**Safety valve:** after roughly 8-10 rounds without convergence, report unresolved findings and the concrete decision or blocker. Do not claim completion.

## Round report format

Keep reports short and traceable:

```
Round N — <clean | N findings | incomplete> — reviewer: <unique name/ID> — revision: <SHA> — clean streak: <0 | 1 | 2>
[if findings, one per line:]
- [severity] file:line — concrete failure → fix applied
```

In the final report, give the total attempted and completed round counts, distinct reviewer identities, incomplete-attempt reasons, findings and fixes, and checks actually executed. Identify the adjacent final clean pair by attempt number and revision. Distinguish a review round from a test run. Do not count earlier runs or self-reviews toward the clean streak.

## Notes

- Keep the review focused on the requested change and concrete regressions; avoid speculative refactors of adjacent flows.
- A previously fixed finding mentioned in the next review is not new unless it remains unresolved or the fix introduced another issue.
- Regression checking requires tracing real usages rather than reading only new modules.
- Run applicable tests as part of the loop. A failing test is a finding to investigate, not a reason to edit the test until the code passes.
