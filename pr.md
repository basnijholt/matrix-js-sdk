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

Defer aggregation for edits until after the event is in the timeline and the thread is initialized, while allowing reactions to aggregate immediately. Concretely:
1. Edits (`RelationType.Replace`): do not aggregate before initialization; queue in `replayEvents` and aggregate after adding to the timeline post-init.
2. Reactions (`RelationType.Annotation`): aggregate immediately even before initialization to keep reaction summaries available; they will be re-aggregated safely on replay (idempotent).

This prevents premature aggregation of edits that creates broken Relations with `targetEvent: null`, while preserving the existing behavior for reactions.

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
        this.replayEvents?.push(event);  // Queue for later

        // Reactions can aggregate immediately (not subject to the edit target lookup race)
        if (event.isRelation(RelationType.Annotation)) {
            this.timelineSet.relations?.aggregateParentEvent(event);
            this.timelineSet.relations?.aggregateChildEvent(event, this.timelineSet);
        }
    } else {
        // Add to timeline...
        
        // Only aggregate AFTER adding to timeline when thread is initialized
        this.timelineSet.relations?.aggregateParentEvent(event);
        this.timelineSet.relations?.aggregateChildEvent(event, this.timelineSet);
    }
}
```

### Why This Works
- Edits that arrive before initialization are queued in `replayEvents` and aggregated only after the target is present in the timeline.
- Reactions aggregate immediately so reaction summaries remain available pre-init; during replay, re-aggregation is a no-op due to deduplication.
- On replay, the thread is initialized and edit aggregation succeeds with proper target linking.

### Idempotence and Deduplication
- `Relations.addEvent` tracks relation event IDs and ignores duplicates, so re-aggregating reactions on replay does not double-count.
- `EventTimelineSet.addEventToTimeline`/`insertEventIntoTimeline` also short-circuit on already-known events, avoiding duplicate timeline entries.

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
- **Risk**: Low — edit aggregation is delayed to a safe point; reaction behavior is preserved and duplicate aggregation is idempotent.

## Notes for Reviewers

The key insight is that edit aggregation must not happen until the thread is initialized and events are in the timeline. The bug was subtle because:
1. It only occurred with specific timing (edits arriving during initialization)
2. The aggregation appeared to work (Relations object created) but was broken internally
3. The 90% success rate made it seem intermittent

The fix ensures edits only aggregate when they can succeed, preventing broken Relations objects. Reactions (e.g. 👍) still aggregate immediately; re-aggregation on replay is safe due to deduplication.

## Checklist

- [x] Tests written for new code (and old code if feasible).
- [x] New or updated `public`/`exported` symbols have accurate [TSDoc](https://tsdoc.org/) documentation.
- [x] Linter and other CI checks pass.
- [x] Sign-off given on the changes (see [CONTRIBUTING.md](https://github.com/matrix-org/matrix-js-sdk/blob/develop/CONTRIBUTING.md)).
