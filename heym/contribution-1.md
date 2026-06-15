# Contribution 1: Log Exceptions from Background Sub-Workflow Execution

**Contribution Number:** 1  
**Student:** Syed Ali Jaseem  
**Issue:** https://github.com/heymrun/heym/issues/197  
**Status:** Pending Review

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

Replace the bare exception handler with a named exception and emit a warning log entry.

This preserves existing fire-and-forget behavior while making failures visible to operators.

### Implementation Plan

Using UMPIRE framework:

**Understand:** Background sub-workflow failures are occurring but exceptions are being silently discarded.

**Match:** Other parts of the codebase already use structured logging to surface runtime failures without interrupting execution flow.

**Plan:**

1. Replace the bare exception handler.
2. Capture the exception object.
3. Emit a warning-level log message containing exception details.
4. Verify parent workflow behavior remains unchanged.
5. Run formatting, linting, and test suites.

**Implement:** Implemented changes on the `fix/drain-bg-futures-silent-exception` branch and submitted PR #198.

**Review:** Verified that fire-and-forget semantics are preserved and that only observability behavior changes.

**Evaluate:** Confirmed that exceptions are now visible in logs while parent workflow execution continues normally.

---

## Testing Strategy

### Unit Tests

- [x] Verify no functional behavior changes to parent workflow execution.
- [x] Verify exception handling logic continues to catch background task failures.

### Integration Tests

- [x] Execute existing backend test suite.
- [x] Confirm no regressions introduced by logging changes.

### Manual Testing

Created a failing background sub-workflow and confirmed that failures now generate warning log entries instead of being silently discarded.

---

## Implementation Notes

### Progress

Reviewed issue #197 and traced background workflow execution to the `drain_bg_futures` function.

Identified a bare exception handler that silently discarded all background execution failures.

Implemented a logging-based solution and submitted PR #198.

### Code Changes

- **Files Modified:** `backend/app/services/workflow_executor.py`
- **Key Pull Request:** https://github.com/heymrun/heym/pull/198
- **Approach Decisions:** Preserved existing fire-and-forget behavior while improving observability through warning-level logging.

---

## Pull Request

**PR Link:** https://github.com/heymrun/heym/pull/198

### PR Description

Replaced a silent exception handler in `drain_bg_futures` with warning-level logging so that failures from background sub-workflows are visible in application logs.

Parent workflow behavior remains unchanged.

### Maintainer Feedback

Pending review.

### Status

Open

---

## Learnings & Reflections

### Technical Skills Gained

- Investigating asynchronous workflow execution
- Improving observability in backend systems
- Understanding failure handling patterns
- Contributing fixes with minimal behavioral impact

### Challenges Overcome

The primary challenge was determining whether exposing exceptions would alter the intended fire-and-forget semantics. After tracing execution flow, it became clear that logging failures could improve observability without affecting workflow behavior.

### What I'd Do Differently Next Time

I would look for additional observability opportunities earlier in the investigation process, such as metrics or tracing, alongside logging improvements.

---

## Resources Used

- Heym Issue #197
- Heym Pull Request #198
- `backend/app/services/workflow_executor.py`
- Project logging patterns
- Backend test suite
