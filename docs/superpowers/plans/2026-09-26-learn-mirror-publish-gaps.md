# Plan: close the three learn.py publish gaps

> Review only. Do not implement until this plan is accepted.
> Target branch stays `atomic-01b-episodic-mirror-fail-closed` (PR #71). No new PR.

**Goal:** One small patch so a failed publish is visible, a later `stage()` for the same id either publishes the kept temp or deletes it, a failed temp unlink is reported, and the mirror line is `fsync`'d under the existing lock.

**Architecture:** Keep the fail-closed order: write temp, `fsync` file, `fsync` directory, append episodic mirror, `os.replace`, `fsync` directory. Do not publish `{cid}.json` on the call where `os.replace` failed. Recovery is the next `stage()` for that same id, plus an `OSError` whose message contains the temp path. `main()` prints that message and exits `1`.

**Files:**

- Modify: `.agent/harness/hooks/_episodic_io.py` (`fsync` before unlock)
- Modify: `.agent/tools/learn.py` (resume, combined cleanup error, directory `fsync`, CLI message)
- Modify: `.agent/tools/test_learn_episodic_mirror.py`

**Out of scope:**

- Renaming `.tmp` to `.json` inside the failing `os.replace` call. That call already failed.
- Teaching `list_candidates`, `graduate.py`, `show.py`, or `auto_dream` to read dotfiles. Graduate opens `{id}.json` only. A dotfile in the review queue would be a candidate those tools cannot accept.
- Tombstones, compensating JSONL, journals, two-phase commit, retry loops, and any new lock. Resume uses the episodic file that already exists and the flock `append_jsonl` already takes.
- Changing candidate id format. `pattern_id` stays a 12-hex MD5.

---

## Why this is the smallest fix

| Gap | What is wrong today | Fix |
|---|---|---|
| 1. Kept temp is invisible | `os.replace` failure leaves `.{cid}.*.tmp`. Readers accept only `*.json`. `stage()` raises a bare `OSError`. Retry appends a second mirror with a new timestamp. | Put the absolute temp path in the `OSError`. On the next `stage()` for that id, either `os.replace` the single valid leftover onto `{cid}.json` with no new mirror, or delete debris. |
| 2. Unlink failure is swallowed | `finally` does `os.remove` and `except OSError: pass`. Mirror failed, temp remains, caller never hears that. | If unlink fails, raise one `OSError` that includes the original error and the temp path. Leave the file on disk. |
| 3. Tests and durability | Fail-closed test stubs `_append_episodic_mirror`. Success tests never check that `.tmp` is gone. `append_jsonl` flushes and does not `fsync`. The candidate directory is not `fsync`'d, so a crash can drop the directory entry. | Real `append_jsonl` failure test. Success test lists the directory. `os.fsync` on the JSONL fd while the flock is still held. `fsync` the candidate directory after the temp name is created and after `os.replace`. |

Rejected: scanning dotfiles from four readers. That is four call sites and still does not graduate the file. Rejected: a journal. The episodic line plus the kept temp is already the record.

---

## Behavior

`cid` is the existing 12-hex `pattern_id`. Temps stay `.{cid}.<random>.tmp` inside `CANDIDATES`, so `os.replace` stays on one filesystem.

### Fresh stage (no leftover)

1. Write the candidate to the temp, `flush`, `fsync` the file, `fsync` the candidate directory.
2. Append the episodic mirror. `append_jsonl` writes, flushes, `fsync`s, then unlocks. It returns only after `fsync`.
3. Set `mirror_succeeded`.
4. `os.replace(temp, {cid}.json)`, `fsync` the directory, set `temp_path = None`.
5. Return `(cid, path)`.

### Publish failure (`os.replace` raises, mirror already returned)

Keep the temp. Raise:

```text
OSError: candidate publish failed; fsynced temp kept at <absolute temp path>
```

Chain the original `OSError` with `raise ... from`. `main()` prints `ERROR: <message>` on stderr and exits `1`. No second mirror on this call.

### Next `stage()` for the same id

Look only at names in `CANDIDATES` that start with `.{cid}.` and end with `.tmp`. Use `os.listdir`, not `glob`.

- **Zero leftovers:** fresh stage.
- **More than one leftover:** raise `OSError` listing every absolute path. Do not write a new temp. Do not append a mirror.
- **Exactly one leftover, and `{cid}.json` already exists:** the visible candidate won. Delete the temp. If delete fails, raise with the path. Do not append.
- **Exactly one leftover, no `{cid}.json`:** load the temp as JSON. Resume only when `id == cid`, `evidence_ids` is a one-element list, and that timestamp is a `"timestamp"` on some line of `AGENT_LEARNINGS.jsonl`. Then `os.replace` that temp onto `{cid}.json`, `fsync` the directory, and return. Do not append another mirror.
- **Exactly one leftover whose JSON is invalid, or whose evidence timestamp is not in the JSONL:** this is debris from a failed mirror (including a failed unlink). Delete it. If delete works, continue with a fresh stage. If delete fails, raise `OSError` with the path and do not publish it. Publishing it would recreate a candidate with a dangling `evidence_id`.

### Mirror or write failure

`mirror_succeeded` stays false. Delete the temp.

- Delete works: re-raise the original error.
- Delete fails: raise

```text
OSError: <original>; failed to remove temp <absolute path>: <unlink error>
```

with `from` the original error. The file stays on disk. Do not turn `KeyboardInterrupt` or `SystemExit` into `OSError`. Only combine when the primary error is an `OSError` (or subclass). For any other `BaseException`, still attempt unlink, then re-raise the original if unlink worked. If unlink failed and the primary is not `OSError`, re-raise the primary anyway so interrupt is not replaced by a cleanup error.

### Directory fsync

```python
def _fsync_dir(directory):
    fd = os.open(directory, os.O_RDONLY)
    try:
        try:
            os.fsync(fd)
        except OSError as err:
            if err.errno not in (errno.EINVAL, errno.ENOTSUP, errno.EBADF, errno.EPERM):
                raise
    finally:
        os.close(fd)
```

If `os.open` on a directory fails with one of those errnos, ignore it. That covers Windows and filesystems with no directory `fsync`. Any other `OSError` propagates and follows the cleanup rules above. Python 3.9 stays supported. Do not use `BaseException.add_note` (3.11+).

### `append_jsonl`

Inside the existing `try`, after `flush` and before the `finally` that unlocks:

```python
os.fsync(f.fileno())
```

`fsync` failure raises. The flock still releases in `finally`. Do not `fsync` from `learn.py` after `append_jsonl` returns. `auto_dream` can rewrite the file as soon as the lock drops, so a late `fsync` can sync the wrong inode contents.

Accepted residual: if `write` succeeds and `fsync` then raises, the line may already be visible and `stage()` will treat the mirror as failed and delete the temp. That is a disk-error path, not a second protocol.

---

## Tasks

### Task 1: Durable append

**File:** `.agent/harness/hooks/_episodic_io.py`

- [ ] After `f.flush()` and before unlock, call `os.fsync(f.fileno())`.
- [ ] Leave the lock, the single `write`, and the return value unchanged.

### Task 2: Resume, cleanup, directory fsync, CLI

**File:** `.agent/tools/learn.py`

- [ ] Add `_fsync_dir`, `_leftover_temps(cid)`, `_evidence_landed(ts)`, and `_remove_or_raise(path, primary)`.
- [ ] At the start of `stage()`, apply the leftover rules above before creating a new temp.
- [ ] On the fresh path, `fsync` the directory after the file `fsync` and again after a successful `os.replace`.
- [ ] On `os.replace` failure, keep the temp and raise the publish message above.
- [ ] On mirror or write failure, stop swallowing `os.remove` errors. Raise the combined message when both fail.
- [ ] In `main()`, catch `OSError` from `stage()` only. Print `ERROR: ...` to stderr and `sys.exit(1)`. Leave the graduation-failure path unchanged.

### Task 3: Tests

**File:** `.agent/tools/test_learn_episodic_mirror.py`

Run from `.agent/tools`:

```bash
python3 -m unittest test_learn_episodic_mirror -v
```

Use `os.listdir` for temp checks so a leading dot cannot hide a file.

- [ ] **Success.** One `*.json`, no name ending in `.tmp`, one episodic row. Patch `hooks._episodic_io.os.fsync` and assert it ran once. Patch `mod._fsync_dir` and assert it ran (file dir entry and the publish).
- [ ] **Real mirror failure.** Keep the real `_append_episodic_mirror`. Make `AGENT_LEARNINGS.jsonl` a directory so `open(..., "ab")` raises `IsADirectoryError`. `stage()` raises. Directory listing has no `.json` and no `.tmp`.
- [ ] **Publish failure.** Existing test, plus the raised message contains the kept temp path. One `.tmp`, no `.json`, one episodic row with `"result": "success"`.
- [ ] **Resume.** After that publish failure, call `stage()` again with the same claim and conditions. Exactly one `.json`, no `.tmp`, still one episodic row. The published `evidence_ids[0]` equals that row's `timestamp`.
- [ ] **Debris is not published.** A single `.tmp` whose evidence timestamp is absent from the JSONL is deleted, then a fresh stage writes one new mirror and one `.json`.
- [ ] **Unlink failure.** Mirror raises `OSError("forced mirror-write failure")` and `os.remove` raises `OSError("forced unlink failure")`. The raised message contains both strings and the temp path. The temp is still on disk. No `.json`.
- [ ] **Ambiguous leftovers.** Two temps for the same id. `stage()` raises, lists both paths, writes no `.json`, and appends no episodic row.

Existing tests `test_stage_writes_one_episodic_mirror`, `test_evidence_id_resolves_to_the_mirror`, and `test_stage_fails_closed_when_mirror_write_errors` stay green. The stubbed fail-closed test can stay. The new directory-as-JSONL test is what covers real `append_jsonl`.

### Task 4: Check

- [ ] `cd .agent/tools && python3 -m unittest test_learn_episodic_mirror -v`
- [ ] All of those tests pass, including the four already on PR #71.
- [ ] Diff is only the three files above.

---

## Crash matrix (after the patch)

| Event | On disk |
|---|---|
| Crash before mirror returns | No JSONL line. Temp name is durable only if the directory `fsync` completed. Next `stage()` deletes a temp with no evidence row, then starts fresh. |
| `os.replace` raises | JSONL line present. Temp kept. Error text has the path. Next `stage()` publishes that temp and does not append again. |
| `os.replace` returns, crash before directory `fsync` | Usual rename window. No new protocol. |
| Unlink fails after mirror failure | Temp kept. Error text says unlink failed and includes the path. Next `stage()` tries delete again because the evidence row is absent. It does not publish that temp. |

---

## Review 5323824586, after this plan shipped on PR #71

Greptile's three P1 comments are not all leftovers of this plan.

**Stays on #71.** `os.fsync` inside shared `append_jsonl` can raise after `flush`. `post_execution.log_execution` and `on_failure` call `append_jsonl` and do not catch `OSError`, so those hook processes die. That regression is in PR #71 (`7a477dd`). Do not move the hook fix onto the idempotent-mirror PR. `append_jsonl_once` is a different function, and the hooks do not call it.

Required #71 behavior: a failed `fsync` in `append_jsonl` must not kill `post_execution` or `on_failure`. Learn's own publish path can still treat a mirror `fsync` failure as an error. One way to keep that split is to catch `OSError` from `fsync` only in the two hook entrypoints, after the line has been written, and leave `append_jsonl`'s raise in place for `stage()`. Do not add a candidate lock to solve this.

**Does not stay on #71.** "Recovery deletes active candidates" and "Recovery misses existing evidence" are episodic-lock work. They are specified in `2026-09-26-learn-mirror-idempotent-append.md`.
