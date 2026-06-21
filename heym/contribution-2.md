# Contribution 2: Prevent Undo/Redo State Corruption During Workflow Execution

**Contribution Number:** 2  
**Student:** Syed Ali Jaseem  
**Issue:** https://github.com/heymrun/heym/issues/199  
**Status:** Complete

---

## Why I Chose This Issue

I chose this issue because it involved state management, workflow execution integrity, and frontend reliability. Although the fix was small, the impact was significant because it prevented users from modifying critical workflow state while execution was actively in progress.

The issue was particularly interesting because it involved the interaction between user actions and a live-running execution engine. It highlighted how seemingly harmless UI actions such as undo and redo can interfere with execution state when proper safeguards are not in place.

---

## Understanding the Issue

### Problem Description

The workflow editor maintained an undo/redo history that allowed users to restore previous versions of the canvas.

However, the `undo()` and `redo()` functions did not check whether a workflow execution was currently running. As a result, users could replace the active canvas state while the execution engine was simultaneously reading from that state.

### Expected Behavior

Undo and redo operations should be blocked while a workflow execution is in progress.

The workflow canvas should remain stable throughout execution and only become editable again after execution completes.

### Current Behavior

The `undo()` and `redo()` functions only checked whether undo or redo history existed.

They immediately replaced `nodes.value` and `edges.value` with a previous history snapshot regardless of execution state.

Since the workflow executor reads the same reactive references, modifying them during execution could cause:

- Incorrect node positions
- Invalid edge connections
- Corrupted execution overlays
- Inconsistent workflow state during execution

### Affected Components

- `frontend/src/stores/workflow.ts`
- `undo()`
- `redo()`
- Workflow execution state management
- Canvas editor state

---

## Reproduction Process

### Environment Setup

I forked the Heym repository, configured the frontend development environment, and reviewed the workflow store implementation responsible for editor history management and execution state.

**Working Branch:**  
https://github.com/syedalijaseem/heym/tree/fix/undo-redo-block-during-execution

### Steps to Reproduce

1. Open a workflow containing several nodes.
2. Make one or more edits so undo history exists.
3. Start workflow execution.
4. While execution is running, press **Ctrl+Z** or **Ctrl+Y**.
5. Observe the workflow canvas.

### Reproduction Evidence

- **Issue:** #199
- **Pull Request:** #200
- **Branch:** `fix/undo-redo-block-during-execution`
- **Root Cause:** `undo()` and `redo()` modified workflow state without checking whether execution was currently in progress.

---

## Solution Approach

### Analysis

The root cause was a missing execution-state guard.

The workflow store already tracked execution status through `isExecuting`, which was correctly updated by `startExecution()` and `stopExecution()`.

However, neither `undo()` nor `redo()` consulted this state before restoring historical snapshots.

### Proposed Solution

Prevent undo and redo operations while workflow execution is active.

The final solution:

- Reuses the existing `isExecuting` state.
- Adds execution-state checks to `undo()`.
- Adds execution-state checks to `redo()`.
- Preserves all existing behavior once execution completes.

This prevents live workflow state from being overwritten while the executor is running.

### Implementation Plan

Using UMPIRE framework:

**Understand:** Undo and redo operations could replace workflow state while execution was active.

**Match:** The application already tracked execution state through `isExecuting`, making it the appropriate mechanism for preventing state changes.

**Plan:**

1. Identify where undo and redo operations are guarded.
2. Extend the existing guards to include execution state.
3. Verify that undo and redo are disabled while workflows are running.
4. Verify that normal undo and redo behavior resumes after execution completes.
5. Run frontend linting and type checking.

**Implement:** Implemented the guard changes on the `fix/undo-redo-block-during-execution` branch and submitted PR #200.

**Review:** Confirmed that the change reused existing execution state instead of introducing new state management logic.

**Evaluate:** Verified that workflow execution remains stable and undo/redo functionality continues working normally after execution finishes.

---

## Testing Strategy

### Unit Tests

- [x] Verify undo operations are blocked while execution is active.
- [x] Verify redo operations are blocked while execution is active.
- [x] Verify normal undo behavior resumes after execution completes.
- [x] Verify normal redo behavior resumes after execution completes.

### Integration Tests

- [x] Run frontend linting.
- [x] Run frontend type checking.
- [x] Confirm no regressions in workflow editor behavior.

### Manual Testing

Performed the following validation steps:

1. Created workflow edits to populate history.
2. Started workflow execution.
3. Attempted undo operations during execution.
4. Attempted redo operations during execution.
5. Confirmed workflow state remained unchanged.
6. Allowed execution to finish.
7. Verified undo and redo worked normally afterward.

---

## Implementation Notes

### Progress

Reviewed issue #199 and traced workflow history management through the workflow store.

Identified that both `undo()` and `redo()` ignored execution state and could overwrite active workflow data during execution.

Implemented execution-state guards using the existing `isExecuting` flag and submitted PR #200.

The change was reviewed, approved, and merged without requiring additional revisions.

### Code Changes

- **Files Modified:** `frontend/src/stores/workflow.ts`
- **Key Pull Request:** https://github.com/heymrun/heym/pull/200
- **Approach Decisions:** Reused existing execution state tracking rather than introducing additional state or synchronization mechanisms.

---

## Pull Request

**PR Link:** https://github.com/heymrun/heym/pull/200

### PR Description

Prevented undo and redo operations from modifying workflow state while execution is in progress.

The change adds execution-state checks to both history operations, ensuring that workflow state remains stable throughout execution while preserving normal undo/redo behavior afterward.

### Maintainer Feedback

The maintainer reviewed and approved the change.

Feedback confirmed that:

- The issue was valid.
- The fix was correct and appropriately scoped.
- Existing execution-state tracking was used effectively.
- The change resolved the workflow state corruption issue.

### Status

Merged

---

## Learnings & Reflections

### Technical Skills Gained

- Understanding reactive state management
- Debugging workflow execution issues
- Preventing invalid state transitions
- Working with frontend stores and execution state
- Contributing targeted fixes to a production application

### Challenges Overcome

The primary challenge was understanding how editor state and execution state interacted during a running workflow.

Although the bug appeared simple, tracing the relationship between undo history, reactive state, and the execution engine was necessary to understand why modifying workflow state during execution caused inconsistent behavior.

### What I'd Do Differently Next Time

I would review user actions that modify shared application state earlier in the investigation process, especially when that state is actively consumed by background execution systems.

---

## Resources Used

- Heym Issue #199
- Heym Pull Request #200
- `frontend/src/stores/workflow.ts`
- Workflow editor state management code
- Frontend linting and type-checking tools
```**
