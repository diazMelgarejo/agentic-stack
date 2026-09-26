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

## Board

| Work | Where | State |
|---|---|---|
| Keep fsynced temp when publish fails after the mirror | PR #71 `0a9f48eb7553a4722e6d1251df7cf29af4596f37` | Pushed. Parent was `30e03cc`. |
| Recover that temp on the next `stage()` without a second mirror, exact timestamp match, read errors are not corrupt JSON | PR #71 `7a477dd39f9081bcace1c47899abcbe716c777fb` | Pushed. 18/18 tests at that commit. Upstream PR body was not rewritten. This checkout cannot edit `codejunkie99/agentic-stack` pull request text. |
| Publish-gap plan | Plan branch `12e2482` parent chain, file `docs/superpowers/plans/2026-09-26-learn-mirror-publish-gaps.md` | Amended this handoff. Hook `fsync` called out as remaining #71 work. |
| Idempotent-mirror plan | Same branch, `docs/superpowers/plans/2026-09-26-learn-mirror-idempotent-append.md` | Amended this handoff. Evidence read must take the existing flock. Not done in code yet. |
| Idempotent `append_jsonl_once` | Fork PR #3 `949aa749c7afa65e7791360ab7fca15c921b4c8c` | Open. Base `atomic-01b-episodic-mirror-fail-closed` at `7a477dd`. 21/21 tests. No candidate lock. |
| Greptile review `5323824586` | Three P1 comments on PR #71 | Routed below. None of them were coded in this last turn. |

## Review routing

Review URL: https://github.com/codejunkie99/agentic-stack/pull/71#pullrequestreview-5323824586

| Comment | Belongs | Why |
|---|---|---|
| Recovery deletes active candidates | Fork PR #3, already covered by mirror-then-temp | #71 writes the temp before the mirror, so a second `stage()` can delete it as unproven. PR #3 appends first. Do not add a `CANDIDATES` lock on #71. |
| Recovery misses existing evidence | Fork PR #3, not implemented | `_evidence_landed` reads the JSONL with no flock while `auto_dream` rewrites it under `LOCK_EX`. The next patch on PR #3 must scan `timestamp` under that same flock before any delete. |
| Sync errors terminate hooks | PR #71, not started | `fsync` was added to shared `append_jsonl` in `7a477dd`. `post_execution.log_execution` and `on_failure` call that function and do not catch `OSError`. PR #3's `append_jsonl_once` is not on their path. |

An older P2 on the same thread, "Mirror can outlive candidate", was already handled on #71 by keeping the temp (`0a9f48e`, then recovery in `7a477dd`).

## What the next agent should do

1. On PR #71 only: stop a failed `append_jsonl` `fsync` from killing `post_execution` and `on_failure`. Leave learn's mirror path able to surface a mirror write error. No new lock.
2. On fork PR #3 only: add `episodic_has_timestamp` (name can differ) in `.agent/harness/hooks/_episodic_io.py`. Hold the existing `LOCK_EX` for the scan. Point `_resumable_evidence` at it. On `OSError` from that read, keep the temp. Extend `test_learn_episodic_mirror`.
3. Do not retarget PR #71 onto the idempotent branch. Do not open a second PR for the evidence-read gap. Push it onto `cursor/mirror-idempotent-append-3352`.
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
