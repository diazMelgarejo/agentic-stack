# Plan: idempotent manual-stage mirror, no second lock

> Original scope is implemented on stacked PR
> https://github.com/diazMelgarejo/agentic-stack/pull/3
> (`949aa74`, base `atomic-01b-episodic-mirror-fail-closed` at `7a477dd`).
> Do not retarget PR #71. One Greptile gap below is still open on that PR.

**Goal:** A second `stage()` for the same candidate id must not append another `manual-stage:{cid}` row. Do that inside the episodic append, under the flock `append_jsonl` and `auto_dream` already share. Do not add a candidate-directory lock, a journal, or a second lock file.

**Why this is separate from PR #71:** PR #71 makes one publish fail-closed and recoverable. It does not make the mirror idempotent. Two callers that both see zero leftovers each call `append_jsonl`, and each gets a new timestamp. A candidate lock would mix two jobs:

| Job | Owner today | This plan |
|---|---|---|
| Episodic append serialization | `fcntl.flock` on `AGENT_LEARNINGS.jsonl` | Same flock. Check-and-append inside it. |
| Candidate transaction coordination | `stage()` temp, `fsync`, `os.replace`, leftover scan | Unchanged. No new lock. |

**Files:**

- Modify: `.agent/harness/hooks/_episodic_io.py` (`append_jsonl_once`)
- Modify: `.agent/tools/learn.py` (use the canonical timestamp; identical temps are not ambiguous)
- Modify: `.agent/tools/test_learn_episodic_mirror.py`

`append_jsonl` stays append-only. Hooks that are supposed to log every event keep calling it.

---

## Mirror API

```python
def append_jsonl_once(path: str, entry: dict, *, match_action: str) -> dict:
```

Open `path` in `"a+b"` so the file is created if missing. Take `LOCK_EX` on that fd for the whole body. On platforms without `fcntl`, skip the lock and keep today's best-effort behavior. Do not add a `threading.Lock` or a sidecar lock file.

While the lock is held:

1. `seek(0)` and read the bytes already in the file.
2. Parse each line as JSON. Skip blank lines and `JSONDecodeError` lines.
3. Return the first object whose `action` equals `match_action` and whose `timestamp` is a non-empty string. Do not append.
4. Otherwise append `entry` as one UTF-8 line, `flush`, `fsync`, and return `entry`.

`O_APPEND` writes go to the end, so a prior `seek(0)` cannot insert in the middle. Unlock in `finally`. Historical duplicate rows stay. This function does not rewrite the file.

The match key is the action string `manual-stage:{cid}`, not the timestamp. File order is the canonical order. The earliest row wins even if a later clock looks smaller.

## What `stage()` changes

`_append_episodic_mirror(cid, claim, source)` builds the entry with a fresh timestamp, calls `append_jsonl_once`, and returns the **returned** row's timestamp. That may be an older row.

Fresh publish order becomes:

```text
resolve leftovers
  → append_jsonl_once          # canonical timestamp, or raise
  → build the candidate with that timestamp
  → write temp, fsync file, fsync directory
  → os.replace
  → best-effort directory fsync
```

The mirror runs before the temp exists. If `append_jsonl_once` raises, there is no temp to delete. A retry calls `append_jsonl_once` again. If the line landed and only `fsync` failed, the retry finds that line and reuses its timestamp. The candidate bytes can be rebuilt because `pattern_id(claim, conditions)` is stable and the timestamp now comes from the row.

`evidence_ids`, `staged_at`, and the staged decision timestamp all use that canonical timestamp. A later `stage()` with a different `source` may rewrite `{cid}.json`, but it must not append.

Leftover recovery still runs first and still does not append. Publish failure still keeps the temp and still reports its path.

## Identical temps

Two callers can both pass the leftover scan, then serialize on the flock and share one timestamp, then each create a temp. That is candidate coordination, and this plan still does not lock it. The follow-up scan treats those twins as one transaction:

- More than one `.{cid}.*.tmp`.
- Every temp is resumable.
- Every evidence timestamp is the same string.

Then publish one (sorted path, existing resume rule against `{cid}.json`) and delete the others. Do not append.

If any temp is corrupt, evidence-less, or carries a different timestamp, keep today's fail-closed `OSError` and do not delete them.

## Tests

`cd .agent/tools && python3 -m unittest test_learn_episodic_mirror -v`

- Two `stage()` calls, no failed replace: one JSONL row, and the published `evidence_ids[0]` equals that row's `timestamp`.
- A pre-existing `manual-stage:{cid}` row with an older timestamp: `stage()` publishes that timestamp and does not append.
- `append_jsonl_once` does not append when the action already exists, and `append_jsonl` still always appends.
- Two temps with the same resumable timestamp: one `.json`, zero `.tmp`, no new row.
- Two temps with different timestamps: `OSError`, both temps remain, row count unchanged.
- Existing fail-closed, publish-failure, and resume tests still pass. A second `stage()` must no longer be expected to create a second mirror.

## Out of scope

- Deleting historical duplicate `manual-stage:{cid}` rows.
- A lock around `CANDIDATES`.
- Cross-process single-flight of temp creation on Windows, where `fcntl` is absent.
- Changing auto-dream's rewrite logic.
- Hook `fsync` failures. Those belong on PR #71. See below.

---

## Greptile review 5323824586

Three P1 comments on PR #71. Routing:

### Already covered here (do not patch #71)

**Recovery deletes active candidates.** On #71, `stage()` writes the temp and then appends the mirror. A second call can see that temp before the row exists, treat it as evidence-less, and delete it. The first call then appends a mirror whose file is gone.

This plan closes that window by appending under the existing flock before any temp exists. PR #3 already does that. A candidate-directory lock on #71 is the wrong fix.

### Still required (not in PR #3 yet)

**Recovery misses existing evidence.** `_evidence_landed` in `learn.py` opens `AGENT_LEARNINGS.jsonl` with no flock. `auto_dream` holds `LOCK_EX`, truncates, and rewrites that file. A retry can observe the truncated file, decide the mirror is absent, delete the temp, and append a new row.

The delete/keep decision has to use the same `LOCK_EX` as `append_jsonl_once`. Add a helper in `.agent/harness/hooks/_episodic_io.py`, for example `episodic_has_timestamp(path, timestamp) -> bool`, that scans parsed `timestamp` fields while the flock is held. `_resumable_evidence` must call that helper. It must not open the JSONL on its own.

If that locked read raises `OSError`, do not delete the temp. A missing proof is not proof of absence when the read itself failed.

Test: the helper flocks before it reads and unlocks after. A temp whose timestamp is present under that lock is not deleted. A temp whose timestamp is absent is still eligible for deletion. Do not simulate this by adding a second lock file.

### Not this plan

**Sync errors terminate hooks.** `fsync` inside shared `append_jsonl` is a PR #71 change. `post_execution` and `on_failure` do not catch `OSError`. Fix that on #71. Do not bury it inside `append_jsonl_once`.
