## Summary

Fixes a race condition where message edits in threads fail to display correctly when they arrive before thread initialization completes. This resolves the issue where Element Web shows the original message instead of the edited version in approximately 10% of cases.

**Fixes**: https://github.com/element-hq/element-web/issues/30617

## The Problem

When edit events arrive while a thread is not yet initialized (`initialEventsFetched = false`), the aggregation system attempts to link the edit to its target message. However, since the thread isn't initialized, the target message may not be in the timeline yet, causing the aggregation to fail silently.

The bug manifests as:
- Edit events are stored in the Relations container
- BUT the `targetEvent` remains `null` because the original message wasn't findable
- Element Web's `replacingEvent()` returns null even though edits exist
- Users see the original message instead of the edited version

## The Solution

Move the aggregation calls inside the `else` block of `addRelatedThreadEvent()` to ensure aggregation only happens when:
1. The thread IS initialized (`initialEventsFetched = true`)
2. The event has been added to the timeline
3. The target event is findable

This prevents premature aggregation that creates broken Relations objects with null target events.

## Technical Details

### Root Cause
In `Thread.addRelatedThreadEvent()`, aggregation was happening unconditionally at the end of the method:
```typescript
private addRelatedThreadEvent(event: MatrixEvent, toStartOfTimeline: boolean): void {
    if (!this.initialEventsFetched) {
        this.replayEvents?.push(event);  // Queue for later
    } else {
        // Add to timeline...
    }
    // BUG: Aggregation happens even when thread not initialized!
    this.timelineSet.relations?.aggregateParentEvent(event);
    this.timelineSet.relations?.aggregateChildEvent(event, this.timelineSet);
}
```

When the thread isn't initialized, the aggregation calls fail to find the target event (it's not in the timeline yet), resulting in a Relations object with `targetEvent: null`.

### The Fix
```typescript
private addRelatedThreadEvent(event: MatrixEvent, toStartOfTimeline: boolean): void {
    if (!this.initialEventsFetched) {
        this.replayEvents?.push(event);  // Queue for later, no aggregation
    } else {
        // Add to timeline...
        
        // Only aggregate AFTER adding to timeline when thread is initialized
        this.timelineSet.relations?.aggregateParentEvent(event);
        this.timelineSet.relations?.aggregateChildEvent(event, this.timelineSet);
    }
}
```

### Why This Works
- Edits that arrive before initialization are queued in `replayEvents`
- When the thread initializes, these events are replayed through `addEvent()`
- On replay, the thread is initialized and aggregation succeeds
- The target event can now be found and properly linked

## Testing

Added comprehensive test that reproduces the race condition:
1. Creates a thread with `initialEventsFetched = false`
2. Adds edit events before the original message
3. Verifies that without the fix, aggregation creates a broken Relations object
4. Confirms that with the fix, aggregation only happens after initialization

The test fails without the fix (showing `targetEvent: null`) and passes with it.

## Impact

- **Bug frequency**: Affected ~10% of edited messages in threads (*in my application that edits messages many times*)
- **User impact**: Messages appeared unedited when they should show edits
- **Scope**: Only affects threads with server-side support enabled
- **Risk**: Low - simply delays aggregation until the proper time

## Notes for Reviewers

The key insight is that aggregation must not happen until the thread is initialized and events are in the timeline. The bug was subtle because:
1. It only occurred with specific timing (edits arriving during initialization)
2. The aggregation appeared to work (Relations object created) but was broken internally
3. The 90% success rate made it seem intermittent

The fix ensures aggregation only happens when it can succeed, preventing the creation of broken Relations objects.

## Checklist

- [x] Tests written for new code (and old code if feasible).
- [x] New or updated `public`/`exported` symbols have accurate [TSDoc](https://tsdoc.org/) documentation.
- [x] Linter and other CI checks pass.
- [x] Sign-off given on the changes (see [CONTRIBUTING.md](https://github.com/matrix-org/matrix-js-sdk/blob/develop/CONTRIBUTING.md)).
