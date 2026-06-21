# Contribution 1: Log Exceptions from Background Sub-Workflow Execution

**Contribution Number:** 1  
**Student:** Syed Ali Jaseem  
**Issue:** https://github.com/heymrun/heym/issues/197  
**Status:** Complete

---

## Why I Chose This Issue

I chose this issue because it involved backend reliability and observability. The bug was small in terms of code changes but had a significant operational impact because exceptions raised by background sub-workflows were being completely discarded.

The issue was interesting because it required understanding asynchronous execution behavior and how failures should be surfaced in production systems. Improving observability without changing application behavior is a common challenge in backend engineering, making this a practical contribution.

---

## Understanding the Issue

### Problem Description

The `drain_bg_futures` function in `backend/app/services/workflow_executor.py` silently discarded exceptions raised by background sub-workflows.

When a background task failed, the exception was caught by a bare `except Exception: pass` block. As a result, failures were completely hidden from operators and developers.

### Expected Behavior

Background sub-workflow failures should be visible in application logs so operators can diagnose issues.

The parent workflow should continue executing successfully because the sub-workflow is intentionally configured as fire-and-forget.

### Current Behavior

Exceptions raised by background sub-workflows are swallowed without any logging, metrics, or traces.

The only indication of a failure is the absence of expected side effects, making debugging difficult.

### Affected Components

- `backend/app/services/workflow_executor.py`
- `drain_bg_futures`
- Background sub-workflow execution
- Application logging and observability

---

## Reproduction Process

### Environment Setup

I forked the Heym repository, configured the local development environment using the project's setup instructions, and reviewed the workflow execution implementation responsible for background task handling.

**Working Branch:**  
https://github.com/syedalijaseem/heym/tree/fix/drain-bg-futures-silent-exception

### Steps to Reproduce

1. Create a workflow containing an **Execute Sub-Workflow (Do Not Wait)** node.
2. Configure the sub-workflow so it fails during execution (e.g., invalid node reference, failing credential, or deliberate exception).
3. Execute the parent workflow.
4. Observe the parent workflow completes successfully.
5. Review application logs.

**Expected:** A warning or error log indicating that the background sub-workflow failed.

**Actual:** No log message is generated and the exception is silently discarded.

### Reproduction Evidence

- **Issue:** #197
- **Pull Request:** #198
- **Branch:** `fix/drain-bg-futures-silent-exception`
- **Root Cause:** A bare `except Exception: pass` block suppresses all exceptions raised by background sub-workflows.

---

## Solution Approach

### Analysis

The root cause is a silent exception handler inside `drain_bg_futures`.

The implementation catches all exceptions and immediately discards them:

```python
try:
    fut.result()
    if done_event is not None:
        done_event.wait()
except Exception:
    pass
```

This prevents failures from being visible in logs and significantly reduces observability of background workflow execution.

### Proposed Solution

Replace the silent exception handler with structured warning-level logging and ensure background workflow completion callbacks are always processed.

The final solution:

- Captures and logs exceptions raised by background sub-workflows.
- Includes traceback information using `exc_info=True`.
- Moves `done_event.wait()` into a `finally` block so callback processing always completes.
- Adds regression tests to verify both logging behavior and error-state persistence.

This preserves existing fire-and-forget behavior while ensuring failures are reliably visible and recorded.

### Implementation Plan

Using UMPIRE framework:

**Understand:** Background sub-workflow failures are occurring but exceptions are being silently discarded.

**Match:** Other parts of the codebase already use structured logging to surface runtime failures without interrupting execution flow.

**Plan:**

1. Replace the bare exception handler.
2. Capture the exception object.
3. Emit a warning-level log message containing exception details.
4. Move `done_event.wait()` into a `finally` block.
5. Add regression tests for logging and error-state persistence.
6. Run formatting, linting, and test suites.

**Implement:** Implemented changes on the `fix/drain-bg-futures-silent-exception` branch and submitted PR #198.

**Review:** Incorporated maintainer feedback regarding callback synchronization, traceback logging, and regression testing.

**Evaluate:** Confirmed that exceptions are now visible in logs, error states are recorded correctly, and parent workflow execution continues normally.

---

## Testing Strategy

### Unit Tests

- [x] Verify warning logs are emitted when a background sub-workflow fails.
- [x] Verify failed background executions are recorded with `status="error"`.
- [x] Verify parent workflow behavior remains unchanged.

### Integration Tests

- [x] Execute full backend test suite (1156 tests).
- [x] Confirm no regressions introduced by the observability changes.

### Manual Testing

Created a failing background sub-workflow and verified:

1. A warning log entry is emitted.
2. Full traceback information is included.
3. The background execution is recorded with `status="error"`.
4. Parent workflow execution continues successfully.

---

## Implementation Notes

### Progress

Reviewed issue #197 and traced background workflow execution to the `drain_bg_futures` function.

Initially identified a bare exception handler that silently discarded all background execution failures and submitted a logging-based fix.

During review, the maintainer identified a race condition where `fut.result()` could raise before callback processing completed, allowing background execution failures to be missed during serialization.

Updated the implementation to:

- Move `done_event.wait()` into a `finally` block.
- Add `exc_info=True` to warning logs.
- Add regression tests covering both logging behavior and error-state persistence.

The updated implementation was approved and merged.

### Code Changes

- **Files Modified:** `backend/app/services/workflow_executor.py`, `backend/tests/test_drain_bg_futures.py`
- **Key Pull Request:** https://github.com/heymrun/heym/pull/198
- **Approach Decisions:** Preserved fire-and-forget semantics while improving observability and guaranteeing background execution status is recorded before serialization.

---

## Pull Request

**PR Link:** https://github.com/heymrun/heym/pull/198

### PR Description

Improved observability and reliability of background sub-workflow execution by replacing a silent exception handler with warning-level logging, ensuring callback completion through a `finally` block, and adding regression tests.

The change preserves existing workflow semantics while making failures visible and ensuring execution status is recorded consistently.

### Maintainer Feedback

The maintainer confirmed the issue was a legitimate observability gap but identified an additional race condition:

- `fut.result()` could raise before callback processing completed.
- Background execution failures could therefore still be missed during serialization.
- Recommended moving `done_event.wait()` into a `finally` block.
- Recommended using `exc_info=True` for full tracebacks.
- Recommended adding regression tests covering both logging and error-state recording.

The implementation was updated accordingly and approved.

### Status

Merged

---

## Learnings & Reflections

### Technical Skills Gained

- Investigating asynchronous workflow execution
- Improving observability in backend systems
- Understanding concurrency and callback synchronization
- Writing regression tests for race-condition scenarios
- Incorporating maintainer feedback into production-quality fixes

### Challenges Overcome

The primary challenge was recognizing that logging the exception alone did not completely resolve the issue. Through maintainer feedback, I learned that callback completion and state persistence also needed to be guaranteed in order to fully eliminate the observability gap.

### What I'd Do Differently Next Time

I would investigate adjacent execution paths earlier in the process to identify related synchronization concerns before submitting the initial pull request.

---

## Resources Used

- Heym Issue #197
- Heym Pull Request #198
- `backend/app/services/workflow_executor.py`
- `backend/tests/test_drain_bg_futures.py`
- Project logging patterns
- Backend test suite
