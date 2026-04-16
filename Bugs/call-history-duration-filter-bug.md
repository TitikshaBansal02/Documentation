# Bug Fix: Call History Duration Filter Empties Table on Apply
 
| Field            | Details                        |
|------------------|--------------------------------|
| **Branch**       | bug-fix                        |
| **Type**         | Bug Fix                        |
| **Status**       | Completed                      |
| **Started**      | 2026-04-16                     |
| **Last Updated** | 2026-04-16                     |
 
---
 
## Overview
 
In the in-app and call history screens, applying the **duration filter** caused all data to disappear instead of filtering correctly. The `isGreaterThan` condition was the primary offender — `isLessThan` accidentally worked, and `isBetween` was unaffected.
 
---
 
## Root Cause
 
The frontend was sending `0` as a placeholder for the unused threshold bound instead of `null` or omitting the field entirely.
 
**Example — user selects "Is Greater Than 120":**
```json
{ "call_duration": { "lower_threshold": 120, "upper_threshold": 0 } }
```
 
The backend schema (`ThresholdFilter` in `call_logs_filter_schema.py:23`) accepts `Optional[int]`, and the repository (`call_logs_repository.py:223–232`) only skips a bound when it is `None` — not when it is `0`:
 
```python
if criteria.call_duration.upper_threshold is not None:   # 0 is not None!
    query = query.filter(CallLogs.duration <= criteria.call_duration.upper_threshold)
```
 
This turned `isGreaterThan 120` into:
```sql
duration >= 120 AND duration <= 0
```
→ no rows ever match → table empties.
 
### Why `isLessThan` accidentally worked
`lower_threshold: 0` produces `duration >= 0`, which is always true — so the filter appeared to work.
 
### Why `isBetween` was unaffected
Both bounds were always set to real user-supplied values.
 
---
 
## Files Changed
 
| File | Location | Change |
|------|----------|--------|
| `useCallRecordFilters.ts` | `revrag-ui/src/hooks/design/` | Lines 164, 168, 237, 241 |
 
---
 
## Fix
 
Two locations in `useCallRecordFilters.ts` were patched — both `filtersState` (useMemo) and `buildFiltersFromValues`:
 
**Before:**
```ts
// isGreaterThan
state.call_duration = { lower_threshold: duration.lower_threshold, upper_threshold: 0 };
 
// isLessThan
state.call_duration = { lower_threshold: 0, upper_threshold: duration.upper_threshold };
```
 
**After:**
```ts
// isGreaterThan
state.call_duration = { lower_threshold: duration.lower_threshold, upper_threshold: null };
 
// isLessThan
state.call_duration = { lower_threshold: null, upper_threshold: duration.upper_threshold };
```
 
Sending `null` causes the backend to skip the unused bound correctly via its `is not None` check.
 
---
 
## Impact Check
 
| Consumer | Handles `null`? | Notes |
|----------|-----------------|-------|
| `call_logs_repository.py:223–232` | ✅ Yes | Skips bound when `None` — this was the intended contract |
| `FilterChips.component.tsx:44–47` | ✅ Yes | Explicitly handles both `0` and `null` — display chips unaffected |
| Existing tests | ✅ Pass | Tests only assert the active bound's value; unused bound not asserted |
 
---
 
## Testing
 
- `isGreaterThan 120` — table now populates with calls longer than 120s ✅
- `isLessThan 60` — continues to work as before ✅
- `isBetween 60 and 180` — unaffected ✅
- Filter chips display correctly for all three modes ✅
---
 
## Notes / Learnings
 
The backend contract was correct — `Optional[int]` with a `None` guard is the right pattern. The bug was purely a frontend misunderstanding of what "no bound" should look like. `0` is a semantically valid duration value (a zero-second call), so using it as a sentinel was always wrong. `null` is the correct way to signal "this bound does not apply."
