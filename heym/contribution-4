# Contribution 4: Prevent Concurrent Dictionary Mutation in Workflow Context Building

**Contribution Number:** 4  
**Student:** Syed Ali Jaseem  
**Issue:** https://github.com/heymrun/heym/issues/219  
**Status:** Complete

---

## Why I Chose This Issue

I chose this issue because it involved concurrency, thread safety, and workflow execution reliability. Unlike many bugs that fail consistently, this issue manifested as an intermittent race condition that could silently break workflow execution under concurrent load.

The issue was particularly interesting because it required analyzing interactions between multiple worker threads and identifying unsafe access to shared state. Concurrency bugs are often difficult to reproduce and diagnose, making this a valuable opportunity to improve the robustness of a production workflow engine.

---

## Understanding the Issue

### Problem Description

The `_build_context` method in `backend/app/services/workflow_executor.py` iterated directly over shared dictionaries used by the workflow executor.

While `_build_context` was reading from these dictionaries, worker threads executing other nodes could simultaneously modify them through `store_node_output`.

This created a race condition where a dictionary could be resized during iteration, resulting in a runtime exception.

### Expected Behavior

`_build_context` should read a consistent snapshot of workflow outputs while constructing execution context.

Concurrent writes from worker threads should not affect an in-progress context build.

### Current Behavior

`_build_context` iterated directly over `self.label_to_output` and checked `self._wrapped_label_output_cache` without acquiring `self.lock`.

At the same time, worker threads called `store_node_output`, which modified both dictionaries while holding the lock.

This created a race condition that could raise:

```text
RuntimeError: dictionary changed size during iteration
```

The exception could terminate the executor thread and leave workflow execution incomplete.

### Affected Components

- `backend/app/services/workflow_executor.py`
- `_build_context`
- `store_node_output`
- Concurrent workflow execution
- Workflow context generation

---

## Reproduction Process

### Environment Setup

I forked the Heym repository, configured the local development environment, and reviewed the workflow executor implementation responsible for context generation and node output storage.

**Working Branch:**  
https://github.com/syedalijaseem/heym/tree/fix/build-context-lock-race

### Steps to Reproduce

1. Create a workflow with multiple nodes capable of executing concurrently.
2. Run the workflow using the executor's thread pool.
3. Allow one thread to enter `_build_context`.
4. Simultaneously allow another thread to call `store_node_output`.
5. Observe intermittent failures during context construction.

### Reproduction Evidence

- **Issue:** #219
- **Pull Request:** #220
- **Branch:** `fix/build-context-lock-race`
- **Root Cause:** `_build_context` iterated over shared dictionaries without synchronization while worker threads could mutate them concurrently.

---

## Solution Approach

### Analysis

The root cause was unsafe iteration over shared mutable state.

The workflow executor already protected writes using `self.lock`, but `_build_context` performed reads without acquiring the same lock.

This meant the code could observe dictionaries while they were being modified, violating thread-safety assumptions.

### Proposed Solution

Take a locked snapshot of both shared dictionaries before beginning any iteration.

The final solution:

- Acquires `self.lock`.
- Copies `self.label_to_output`.
- Copies `self._wrapped_label_output_cache`.
- Releases the lock immediately after the snapshots are created.
- Uses only snapshot data throughout context construction.

This guarantees a consistent view of workflow outputs while minimizing lock contention.

### Implementation Plan

Using UMPIRE framework:

**Understand:** `_build_context` was reading shared dictionaries concurrently with writer threads.

**Match:** Existing code already used `self.lock` to protect writes, indicating that shared state access required synchronization.

**Plan:**

1. Acquire `self.lock`.
2. Create snapshots of both shared dictionaries.
3. Release the lock immediately.
4. Replace all reads of live dictionaries with snapshot reads.
5. Add a regression test that repeatedly executes `_build_context` while another thread continuously updates outputs.
6. Run formatting, linting, and backend test suites.

**Implement:** Implemented the snapshot-based approach on the `fix/build-context-lock-race` branch and submitted PR #220.

**Review:** Verified that lock duration remained minimal and that workflow behavior was unchanged.

**Evaluate:** Confirmed that concurrent execution no longer triggers dictionary mutation errors.

---

## Testing Strategy

### Unit Tests

- [x] Verify `_build_context` can execute while outputs are being updated concurrently.
- [x] Verify no `RuntimeError` occurs during dictionary iteration.
- [x] Verify workflow context generation remains correct.

### Integration Tests

- [x] Execute full backend test suite (1291 tests).
- [x] Confirm no regressions introduced by the synchronization change.

### Manual Testing

Created concurrent execution scenarios involving repeated calls to `_build_context` while worker threads continuously updated node outputs.

Verified that context generation completed successfully without runtime exceptions.

---

## Implementation Notes

### Progress

Reviewed issue #219 and traced workflow context generation through `_build_context`.

Identified unsafe iteration over shared dictionaries that could be modified concurrently by worker threads.

Implemented a snapshot-based solution that acquires the lock only long enough to copy shared state.

Added a concurrency regression test to reproduce and validate the fix.

After review and CI testing, the change was approved and merged.

### Code Changes

- **Files Modified:** `backend/app/services/workflow_executor.py`
- **Key Pull Request:** https://github.com/heymrun/heym/pull/220
- **Approach Decisions:** Used snapshot-based reads rather than holding locks during context generation to maximize safety while minimizing contention.

---

## Pull Request

**PR Link:** https://github.com/heymrun/heym/pull/220

### PR Description

Prevented concurrent dictionary mutation during workflow context construction by creating locked snapshots of shared executor state before iteration begins.

Added regression coverage to verify that concurrent writes cannot trigger runtime failures during context generation.

### Maintainer Feedback

The maintainer reviewed and approved the implementation.

Feedback confirmed that:

- The race condition was valid.
- The snapshot-based approach was appropriate.
- Lock duration remained minimal.
- The regression test successfully validated the fix.

### Status

Merged

---

## Learnings & Reflections

### Technical Skills Gained

- Diagnosing race conditions in multi-threaded systems
- Understanding thread-safe access patterns for shared state
- Using snapshot-based synchronization techniques
- Writing concurrency-focused regression tests
- Contributing fixes to a production workflow engine

### Challenges Overcome

The primary challenge was reasoning about a bug that occurred only under specific timing conditions. Because concurrent mutations depended on thread scheduling, the issue was intermittent and difficult to reproduce consistently.

By tracing read and write access patterns to shared dictionaries, I was able to identify the synchronization gap and implement a deterministic fix.

### What I'd Do Differently Next Time

I would review shared-state access patterns earlier when investigating workflow execution code, especially in areas that interact with thread pools or asynchronous execution.

---

## Resources Used

- Heym Issue #219
- Heym Pull Request #220
- `backend/app/services/workflow_executor.py`
- Project concurrency and workflow execution code
- Backend test suite
