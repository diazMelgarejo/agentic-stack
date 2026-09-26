# Plan: idempotent manual-stage mirror, no second lock

> Review only until accepted. Stack the code on PR #71
> (`atomic-01b-episodic-mirror-fail-closed`). Do not retarget #71.

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
