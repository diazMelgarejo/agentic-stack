# Coordination board handoff

## Agent envelope

```text
agent_id: bc-39be2f33-6309-588f-b74c-3b95331d3352
agent_url: https://cursor.com/agents/bc-39be2f33-6309-588f-b74c-3b95331d3352
run_name: agentic-stack #71 P2 keep-temp after mirror
model: grok-4.7
source: sand
owning_user: seth Nimbosa
owning_user_email: darth.serious@gmail.com
repo: https://github.com/diazMelgarejo/agentic-stack
upstream_pr: https://github.com/codejunkie99/agentic-stack/pull/71
date: 2026-09-26
subagent_reviewer_id: bc-1ff302be-b9af-5606-a036-5d00759509a5
```

There is no upstream pull request #72. Issue #72 is the closed GitHub App write-access note. The follow-up pull request is on the fork: https://github.com/diazMelgarejo/agentic-stack/pull/3

This file is the same handoff, updated in place. The raw URL did not change.

## Board

| Work | Where | State |
|---|---|---|
| Keep fsynced temp when publish fails after the mirror | PR #71 `0a9f48eb7553a4722e6d1251df7cf29af4596f37` | Pushed. Parent was `30e03cc`. |
| Recover that temp on the next `stage()` without a second mirror, exact timestamp match, read errors are not corrupt JSON | PR #71 `7a477dd39f9081bcace1c47899abcbe716c777fb` | Pushed. 18/18 tests at that commit. Upstream PR body was not rewritten. This checkout cannot edit `codejunkie99/agentic-stack` pull request text. |
| Publish-gap plan | Plan branch, `docs/superpowers/plans/2026-09-26-learn-mirror-publish-gaps.md` | Historical plan. Status note at the end points at the landed SHAs. |
| Idempotent-mirror plan | Same branch, `docs/superpowers/plans/2026-09-26-learn-mirror-idempotent-append.md` | Historical plan. Status note at the end points at the landed SHAs. |
| Idempotent `append_jsonl_once` | Fork PR #3 `949aa749c7afa65e7791360ab7fca15c921b4c8c` | Pushed. Base `atomic-01b-episodic-mirror-fail-closed` at `7a477dd`. |
| Locked timestamp scan | Fork PR #3 `be684ff8de6bdcfd69ff325405a7f93c6783ad19` | Pushed. `has_jsonl_timestamp` holds `LOCK_EX` and does not create a missing JSONL. |
| Hook `OSError` catch | Fork PR #3 `409e7417815bdd00f7fced6cb8ae486df476f856` | Pushed on the stacked branch, not on PR #71. The user overrode the earlier "#71 only" routing. `append_jsonl` still raises. |
| Greptile review `5323824586` | Three P1 comments on PR #71 | All three have a fix SHA. Upstream inline replies returned HTTP 403. SHAs are on the fork PR comment below. Threads were not resolved. |

HEAD of `cursor/mirror-idempotent-append-3352` is `409e7417815bdd00f7fced6cb8ae486df476f856`. Tests at that tip: `test_learn_episodic_mirror` 24/24, `test_episodic_hooks` 2/2.

## Review routing

Review URL: https://github.com/codejunkie99/agentic-stack/pull/71#pullrequestreview-5323824586

| Comment | Fix SHA | What landed |
|---|---|---|
| 4109582726 Recovery deletes active candidates | `949aa749c7afa65e7791360ab7fca15c921b4c8c` | `stage()` appends the mirror before any temp exists. No `CANDIDATES` lock. |
| 4109582730 Recovery misses existing evidence | `be684ff8de6bdcfd69ff325405a7f93c6783ad19` | `has_jsonl_timestamp` scans parsed `timestamp` fields under the existing episodic `LOCK_EX`. Missing file is absence (`rb` open, `FileNotFoundError` returns false). Any other `OSError` propagates, so a resumable temp is kept. |
| 4109582734 Sync errors terminate hooks | `409e7417815bdd00f7fced6cb8ae486df476f856` | `post_execution.log_execution` and `on_failure.on_failure` catch `OSError`, warn on stderr, and return the entry. `append_jsonl` itself still raises, so learn stays fail-closed. |

The uploaded patch opened the JSONL with `a+b` and called `makedirs`. That would create the episodic file on a pure existence check. The landed helper does not do that.

An older P2 on the same pull request, "Mirror can outlive candidate", was already handled on #71 by keeping the temp (`0a9f48e`, then recovery in `7a477dd`). That thread is already resolved.

Fork record of the three SHAs: https://github.com/diazMelgarejo/agentic-stack/pull/3#issuecomment-5841869809

Earlier routing comment: https://github.com/diazMelgarejo/agentic-stack/pull/3#issuecomment-5841747832

## What the next agent should do

1. Do not reimplement the three Greptile P1s. They are on `cursor/mirror-idempotent-append-3352`.
2. If write access to `codejunkie99/agentic-stack` appears, reply on comments 4109582726, 4109582730, and 4109582734 with the SHAs in the table, then resolve those three threads. Do not resolve a thread that has no SHA reply.
3. Do not retarget PR #71 onto the idempotent branch. Do not merge. Do not force-push.
4. Downloads for a human must be `refs/heads` raw URLs. A path on the agent machine is not a download.

## Links

Plan, publish gaps:

https://raw.githubusercontent.com/diazMelgarejo/agentic-stack/refs/heads/cursor/plan-learn-publish-gaps-3352/docs/superpowers/plans/2026-09-26-learn-mirror-publish-gaps.md

Plan, idempotent mirror:

https://raw.githubusercontent.com/diazMelgarejo/agentic-stack/refs/heads/cursor/plan-learn-publish-gaps-3352/docs/superpowers/plans/2026-09-26-learn-mirror-idempotent-append.md

This handoff:

https://raw.githubusercontent.com/diazMelgarejo/agentic-stack/refs/heads/cursor/plan-learn-publish-gaps-3352/docs/superpowers/plans/2026-09-26-coordination-board-handoff.md

PR #71: https://github.com/codejunkie99/agentic-stack/pull/71

Fork PR #3: https://github.com/diazMelgarejo/agentic-stack/pull/3

Review canvas from this run: `/cursor/stores/user/canvases/19a7433b-17e3-4493-9e2a-f514749acb17/source.canvas.tsx` (Cursor-only path, not a GitHub download).
