# Contribution 3: Prevent Expired or Invalid HITL Requests from Resuming Workflow Execution

**Contribution Number:** 3  
**Student:** Syed Ali Jaseem  
**Issue:** https://github.com/heymrun/heym/issues/217  
**Status:** Phase IV Complete

---

## Why I Chose This Issue

I chose this issue because it involved workflow correctness, background task reliability, and defensive programming. Human-in-the-Loop (HITL) workflows depend on strict approval and decision lifecycles, making it important that expired or invalid requests cannot resume workflow execution.

The issue was particularly interesting because the initial proposed fix appeared correct but revealed a deeper understanding of the actual HITL lifecycle during code review. The contribution became an opportunity to learn how production workflows transition through different states and how seemingly reasonable assumptions can be incorrect when compared against real application behavior.

---

## Understanding the Issue

### Problem Description

The `resume_hitl_request_in_background` function in `backend/app/services/hitl_service.py` resumed workflow execution without validating whether the associated HITL request was still in a valid state.

A delayed or stale background task could potentially attempt to resume workflow execution after a request had expired or otherwise become invalid.

### Expected Behavior

Only valid, resolved HITL requests with an associated decision should be allowed to resume workflow execution.

Expired, unresolved, or otherwise invalid requests should be ignored.

### Current Behavior

`resume_hitl_request_in_background` loaded a `HITLRequest` and proceeded toward workflow resumption without validating the request lifecycle state.

The function only checked whether the request existed:

```python
if hitl_request is None:
    return
```

No additional validation occurred before workflow execution resumed.

### Affected Components

- `backend/app/services/hitl_service.py`
- `resume_hitl_request_in_background`
- HITL request lifecycle management
- Background workflow resumption

---

## Reproduction Process

### Environment Setup

I forked the Heym repository, configured the backend development environment, and reviewed the HITL request lifecycle implementation along with the workflow resumption path.

**Working Branch:**  
https://github.com/syedalijaseem/heym/tree/fix/hitl-resume-expiry-guard

### Steps to Reproduce

1. Create a workflow containing a HITL node.
2. Execute the workflow to generate a pending HITL request.
3. Allow the request to become stale or invalid before resumption.
4. Trigger `resume_hitl_request_in_background`.
5. Observe workflow execution continues without validating request state.

### Reproduction Evidence

- **Issue:** #217
- **Pull Request:** #218
- **Branch:** `fix/hitl-resume-expiry-guard`
- **Root Cause:** The background resume path lacked validation of the HITL request lifecycle before resuming workflow execution.

---

## Solution Approach

### Analysis

The original investigation focused on expired requests and proposed preventing workflow resumption when requests were no longer pending.

During maintainer review, it became clear that the actual HITL lifecycle differs from the initial assumption.

The application transitions requests from:

```text
pending → resolved
```

before scheduling background workflow resumption.

As a result, blocking all non-pending requests would accidentally prevent valid workflow execution from resuming.

The true requirement was ensuring that only properly resolved requests with a valid decision could proceed.

### Proposed Solution

Add defensive validation to the background resume path that permits only valid resolved requests.

The final solution:

- Allows only requests with `status="resolved"`.
- Requires a valid decision value.
- Blocks pending requests.
- Blocks expired requests.
- Blocks unresolved requests.

This aligns background execution with the actual HITL lifecycle used throughout the application.

### Implementation Plan

Using UMPIRE framework:

**Understand:** Background workflow resumption lacked lifecycle validation.

**Match:** Other HITL code paths already validated request state before performing actions.

**Plan:**

1. Analyze the HITL lifecycle implementation.
2. Add lifecycle validation before workflow resumption.
3. Verify that valid resolved requests continue to work.
4. Add regression tests covering valid and invalid states.
5. Run formatting, linting, and backend test suites.

**Implement:** Implemented the initial guard and submitted PR #218.

**Review:** Incorporated maintainer feedback after discovering the actual lifecycle behavior differed from the initial assumptions.

**Evaluate:** Verified that valid workflow resumptions continue to work while invalid requests are safely rejected.

---

## Testing Strategy

### Unit Tests

- [x] Verify resolved requests with valid decisions are allowed to resume.
- [x] Verify pending requests are rejected.
- [x] Verify expired requests are rejected.
- [x] Verify resolved requests without decisions are rejected.

### Integration Tests

- [x] Execute full backend test suite.
- [x] Confirm no regressions introduced by lifecycle validation.

### Manual Testing

Verified that valid HITL approval flows continue to resume workflow execution normally.

Also verified that invalid request states are blocked before workflow resumption occurs.

---

## Implementation Notes

### Progress

Reviewed issue #217 and traced execution through the HITL workflow resumption path.

Initially implemented a guard that blocked requests that were not pending or had already expired.

During code review, the maintainer identified that the application marks requests as `resolved` before scheduling background workflow resumption. This meant the original guard would incorrectly block valid workflow execution.

Updated the implementation to validate the actual lifecycle state used by the application.

Reworked the regression tests to match real HITL status values and decision behavior.

The updated implementation was approved and merged.

### Code Changes

- **Files Modified:** `backend/app/services/hitl_service.py`, `backend/tests/test_hitl_resume_expiry_guard.py`
- **Key Pull Request:** https://github.com/heymrun/heym/pull/218
- **Approach Decisions:** Matched the actual HITL lifecycle rather than relying on assumptions about request status transitions.

---

## Pull Request

**PR Link:** https://github.com/heymrun/heym/pull/218

### PR Description

Added defensive validation to `resume_hitl_request_in_background` so that workflow execution only resumes for valid resolved HITL requests with decisions.

Added regression tests covering both valid and invalid lifecycle states.

### Maintainer Feedback

The maintainer confirmed the underlying issue was valid but identified a flaw in the initial implementation.

Key feedback included:

- The application transitions requests from `pending` to `resolved` before scheduling background resumption.
- Blocking all non-pending requests would prevent valid workflow execution.
- Status values such as `accepted` and `refused` were not actual `HITLRequest.status` values.
- Tests should validate the real lifecycle used by the application.
- The guard should allow resolved requests with decisions while rejecting invalid states.

The implementation and tests were updated accordingly.

### Status

Merged

---

## Learnings & Reflections

### Technical Skills Gained

- Understanding workflow approval lifecycles
- Investigating asynchronous background execution
- Writing defensive validation logic
- Revising assumptions through code review feedback
- Creating regression tests for workflow state transitions

### Challenges Overcome

The primary challenge was recognizing that the initial fix, while seemingly correct, did not match the application's actual lifecycle.

Through maintainer feedback and additional investigation, I learned that understanding real production state transitions is more important than relying on assumptions derived from issue descriptions alone.

### What I'd Do Differently Next Time

I would trace the complete lifecycle of state transitions earlier in the investigation process before proposing validation logic.

This would help ensure that fixes align with actual application behavior from the outset.

---

## Resources Used

- Heym Issue #217
- Heym Pull Request #218
- `backend/app/services/hitl_service.py`
- `backend/tests/test_hitl_resume_expiry_guard.py`
- HITL workflow lifecycle implementation
- Backend test suite
