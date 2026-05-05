# Backup Issue Analysis — Step 02: Journal Log Evidence

## Summary

Analysis of `2026-05-05-journal.log` **confirms Bug 2 (±5 min validation window)** as the
**direct cause** of the missed backup, with Bug 1 (missing `stop()`) as a **contributing factor**.

The backup was scheduled for **00:16**, but the `BackgroundActivity` timer fired at **00:32**
(16 minutes late). The `isSyncScheduled()` check rejected it as "wrong wake up" because
00:32 is outside the ±5 minute tolerance around 00:16.

---

## Timeline of Events

### Phase 1: Manual backup trigger (23:38)

| Time | Event |
|------|-------|
| 23:38:03 | Various profiles rescheduled (carddav, caldav, Images, Posts) |
| 23:38:34 | `BackgroundActivity` fires for `nextcloud.Images-28` and `nextcloud.Posts-28` |
| 23:38:34 | **Both rejected**: *"Woken up of nextcloud.Images-28 in a disabled period"* |
| 23:38:34 | **Both rejected**: *"Woken up of nextcloud.Posts-28 in a disabled period"* |
| 23:38:38 | Manual backup starts for `nextcloud.Backup-28` |
| 23:38:38 | `BackgroundSync::set()` — **"Rescheduling for nextcloud.Backup-28 with frequency 37 minutes"** |
| 23:38:38 | `nextSyncTime` calculated: **QDateTime(2026-05-05 00:16:00.000 CEST)** |
| 23:38:38 | 37 min to 00:16 → maps to `OneHour` frequency bucket |

### Phase 2: Backup executes and completes (23:38–23:39)

| Time | Event |
|------|-------|
| 23:38:38 | Backup starts (BackupQuery, then actual backup) |
| 23:38:42 | Backup archive creation begins |
| 23:39:02 | tar file created, upload starts |
| 23:39:07 | **Backup finished successfully** (status: 4) |
| 23:39:07 | Session cleaned up for `nextcloud.Backup-28` |

### Phase 3: Post-completion rescheduling (23:39)

| Time | Event |
|------|-------|
| 23:39:36 | `setNextAlarm()` called for `nextcloud.Backup-28` |
| 23:39:36 | lastSync = `2026-05-04 21:39:07 UTC` |
| 23:39:36 | **"Explicit sync time defined"** |
| 23:39:36 | next sync = **QDateTime(2026-05-05 00:16:00.000 CEST)** ← correct! |
| 23:39:36 | `BackgroundSync::set()` → **"Frequency unchanged for nextcloud.Backup-28, waiting."** ← Bug 1 path (no `stop()`) |

**Key observation:** The `BackgroundActivity` already existed from the pre-backup scheduling
at 23:38:38 with `OneHour` frequency. Now at 23:39:36, the frequency is still `OneHour`
(~37 min to 00:16 → same bucket), so the code takes the "unchanged" path and calls
`wait()` without `stop()`. Since the state was already `Waiting`, the `wait()` call is
a **no-op** — the timer continues with its original schedule from 23:38:38.

### Phase 4: False trigger and rejection (23:40)

| Time | Event |
|------|-------|
| 23:40:07 | *"Triggering queued profile modification sync for nextcloud.Backup-28"* |
| 23:40:07 | `startScheduledSync()` called |
| 23:40:07 | **"Woken up of nextcloud.Backup-28 in a disabled period, not starting sync."** |

**Why rejected:** 23:40 is NOT within ±5 min of 00:16 (the scheduled time).
This is a profile modification trigger, not the timer — but same validation applies.

### Phase 5: Timer fires — too late! (00:32)

| Time | Event |
|------|-------|
| 00:32:51 | **`BackgroundSync` fires for `nextcloud.Backup-28`** |
| 00:32:51 | *"Check if sync is scheduled against t kvě 5 00:32:51 2026"* |
| 00:32:51 | **"Woken up of nextcloud.Backup-28 in a disabled period, not starting sync."** |
| 00:32:51 | *"Sync cancelled due to wrong wake up"* (status: 8, ABORTED) |
| 00:32:51 | Activity removed, new alarm set |
| 00:32:51 | lastSync = `2026-05-04 21:39:07 UTC` |
| 00:32:51 | **next sync = QDateTime(2026-05-06 00:16:00.000 CEST)** ← skipped a whole day! |
| 00:32:51 | `BackgroundSync::set()` — *"profile name = nextcloud.Backup-28 with frequency 1423 minutes"* |

**Root cause:** The scheduled time was 00:16, but the timer fired at 00:32 =
**16 minutes late**. The `isSyncScheduled()` check uses a ±5 minute window:

```
00:16 - 5min = 00:11  (earliest allowed)
00:16 + 5min = 00:21  (latest allowed)
00:32         = REJECTED (16 min late)
```

After rejection, `nextSyncTime()` calculates next sync as **May 6th** at 00:16 —
the backup for May 5th is completely skipped.

---

## Why Was the Timer 16 Minutes Late?

Two factors combined:

1. **Bug 1 (missing `stop()`):** At 23:39:36, the rescheduling took the "frequency unchanged"
   path and called `wait()` without `stop()`. Since the `BackgroundActivity` was already in
   `Waiting` state (set at 23:38:38), the `wait()` was effectively ignored. The timer
   continued counting from its original schedule set at 23:38:38.

2. **Frequency quantization:** The time gap from 23:38:38 to 00:16 is ~37 minutes, which
   maps to `OneHour` frequency bucket. The `BackgroundActivity` with `OneHour` frequency
   doesn't guarantee wakeup at exactly the right time — it woke up at 00:32:51, roughly
   54 minutes after the 23:38:38 scheduling, which is within the OneHour bucket range but
   16 minutes past the target of 00:16.

---

## Confirmed Findings

| Bug | Status | Evidence |
|-----|--------|----------|
| Bug 1 — Missing `stop()` | **CONFIRMED** | Line 2710: `"Frequency unchanged for nextcloud.Backup-28, waiting."` — timer not actually reset |
| Bug 2 — ±5 min window | **CONFIRMED as direct cause** | Line 12333: `"Woken up of nextcloud.Backup-28 in a disabled period"` at 00:32, scheduled for 00:16 |
| Bug 3 — Frequency quantization | **CONFIRMED** | 37 min → OneHour bucket → fires at 00:32 instead of 00:16 |
| "disabled period" for other profiles | **Also observed** | Lines 1018, 1051: Images-28 and Posts-28 also rejected as "disabled period" at 23:38:34 |

---

## Also Affected: Other Sync Profiles

The log also shows `nextcloud.Images-28` and `nextcloud.Posts-28` being repeatedly
rejected as "disabled period" throughout the log:

- 23:38:34 — Images-28 and Posts-28 both rejected
- 00:32:51 — Both rejected again
- 01:32:53 — Images-28 rejected
- 01:37:51 — Images-28 rejected again

This confirms the ±5 minute window issue is a **systematic problem** affecting multiple
sync profiles, not just backups.

---

## Conclusions

The missed backup is caused by a **chain of two bugs**:

1. **Bug 1** (missing `stop()` before `wait()`) — prevents proper timer reset, causing
   the `BackgroundActivity` to fire at an imprecise time determined by the frequency bucket
   rather than the intended target time.

2. **Bug 2** (±5 minute validation window) — rejects the sync when the timer fires more
   than 5 minutes from the scheduled time. For daily backups with explicit time schedules,
   this window is far too tight given the imprecision of frequency-based `BackgroundActivity`.

**Both bugs need to be fixed for reliable backup scheduling:**

- **Bug 1 fix:** Add `stop()` before `wait()` in the "frequency unchanged" path of
  `BackgroundSync::set()`. This alone may help if the timer fires closer to the target
  but is insufficient when quantization drift exceeds 5 minutes.

- **Bug 2 fix:** Either widen the tolerance window significantly (e.g., to ±30 minutes
  or half the scheduling interval), or skip the `isSyncScheduled()` validation for
  scheduler-triggered syncs (since the scheduler already determined the correct time).

A more robust fix would be to use the one-shot `wait(seconds)` path for explicit-time
schedules instead of the frequency-based path, since the exact target time is known.

