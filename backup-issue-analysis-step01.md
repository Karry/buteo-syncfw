# Backup Issue Analysis — Step 01

## Summary

Scheduled backup on Sailfish OS sometimes stops working after the first successful run.
The `msyncd` daemon (buteo-syncfw) is responsible for scheduling and executing backups
using `BackgroundActivity` from nemo-keepalive.

Analysis of the source code reveals several bugs in the scheduling lifecycle that can
cause subsequent backup triggers to silently fail.

---

## Bug 1 (Primary Suspect): Missing `stop()` before `wait()` in `BackgroundSync::set()`

**File:** `msyncd/BackgroundSync.cpp`, lines 109–112

When rescheduling after a sync completes, if the frequency bucket hasn't changed
(which is the typical case for daily backups), the code calls `wait()` **without
first calling `stop()`**:

```cpp
} else {
    newAct.backgroundActivity->wait();  // BUG: no stop() before wait()
    qCDebug(lcButeoMsyncd) << "BackgroundSync::set() Frequency unchanged for" << aProfName << ", waiting.";
    return true;
}
```

The `setSwitch()` method in the same file (line 255–256) explicitly documents why
this is wrong:

> *"If activity's state was already Waiting, the state doesn't change, nothing
> happens and the existing background activity keeps running until the previously
> set time expires, so we have to stop it."*

`setSwitch()` correctly calls `stop()` before `wait()`, but `set()` does not in
the unchanged-frequency branch. This means **after the first backup completes, the
timer is never actually reset**, so subsequent wakeups never fire.

**This directly explains the reported symptom: "automatic daily backup only works
once after creation."**

### Fix

Add `newAct.backgroundActivity->stop()` before the `wait()` call on line 110:

```cpp
} else {
    newAct.backgroundActivity->stop();
    newAct.backgroundActivity->wait();
    qCDebug(lcButeoMsyncd) << "BackgroundSync::set() Frequency unchanged for" << aProfName << ", waiting.";
    return true;
}
```

### How to Confirm

```bash
# Enable debug logging and watch the journal after a backup completes:
journalctl --user -u msyncd -f | grep -E "(BackgroundSync::set|Frequency unchanged|Rescheduling|BackgroundSync started|BackgroundSync completed)"
```

**What to look for:** After backup completes, you should see
`"Frequency unchanged for ... waiting."` — this means `wait()` was called without
`stop()`, so the timer likely wasn't reset. If `"BackgroundSync started"` never
appears again the next day, this confirms the bug.

**Quick workaround test:** Restart msyncd after each backup:
```bash
systemctl restart --user msyncd
```
If the next day's backup works after a restart but not without it, this strongly
confirms Bug 1.

---

## Bug 2: Tight ±5 Minute Time Validation Window in `isSyncScheduled()`

**File:** `libbuteosyncfw/profile/SyncSchedule.cpp`, lines 513–514

When `BackgroundActivity` fires, `startScheduledSync()` (`synchronizer.cpp:312`)
validates the wakeup time against the profile's configured schedule. For explicit-time
schedules (e.g., "backup at 02:00"), it checks:

```cpp
return (aActualDateTime.time() < d_ptr->iTime.addSecs(5 * 60)
        && aActualDateTime.time() > d_ptr->iTime.addSecs(-5 * 60));
```

If the `BackgroundActivity` timer drifts by more than 5 minutes (due to system
suspend/resume, power management delays, or frequency bucket quantization from Bug 3),
the sync is **silently rejected** with:

> *"Woken up of \<profile\> in a disabled period, not starting sync."*

### Fix

Widen the tolerance or skip this check for scheduler-triggered syncs (since the
scheduler itself already validated the timing when it set the alarm).

### How to Confirm

```bash
journalctl --user -u msyncd -f | grep -i "disabled period"
```

If the message *"Woken up of \<profile\> in a disabled period, not starting sync"*
appears, the timer fired outside the ±5 min window.

---

## Bug 3: Frequency Quantization Creates Boundary Issues for Daily Backups

**File:** `msyncd/BackgroundSync.cpp`, lines 94 and 122

The threshold for switching between frequency-based and one-shot timers is
`seconds / 60 > MAX_FREQUENCY` where `MAX_FREQUENCY = 1440` (24 hours).

A daily backup interval of exactly 86400 seconds (= 1440 minutes) does **not**
satisfy `> 1440`, so it uses frequency-based scheduling with
`BackgroundActivity::TwentyFourHours`. The one-shot path (`wait(seconds)`) would be
more precise.

Due to computation timing, `secsTo(nextSyncTime) + 1` (see `SyncScheduler.cpp:227`)
can fluctuate around the boundary — sometimes 1439 minutes (→ frequency-based),
sometimes 1441 minutes (→ one-shot) — causing **inconsistent scheduling behavior**
between runs.

### Fix

Use `>=` instead of `>` to consistently pick the one-shot path for 24h intervals:

```cpp
if (seconds / 60 >= MAX_FREQUENCY) {
```

### How to Confirm

```bash
journalctl --user -u msyncd -f | grep -E "(without a valid frequency|with frequency)"
```

For daily backups, watch whether you see `"without a valid frequency, waiting for N
seconds"` (one-shot) or `"with frequency 1440 minutes"` (quantized). Inconsistency
between runs indicates the boundary issue.

---

## Bug 4: Double-Scheduling Race Condition in `syncStatusChanged()`

**File:** `msyncd/SyncScheduler.cpp`, lines 164–184

After a sync completes, two rescheduling paths execute:

1. `cleanupSession()` → `reschedule()` → `addProfile()` → `set()` — creates/updates
   the background activity
2. Via `Qt::QueuedConnection`: `slotSyncStatus()` → `syncStatusChanged()` →
   `onBackgroundSyncCompleted()` → `remove()` **destroys** that activity, then
   `setNextAlarm()` → `set()` creates a **new** one

If `iProfileManager.syncProfile(aProfileName)` returns `nullptr` at line 177
(e.g., profile temporarily not loadable from disk), `setNextAlarm` is skipped and
**the schedule is permanently lost** until msyncd is restarted.

### Fix

Avoid double-scheduling by not rescheduling in `syncStatusChanged()` when it was
already done via `reschedule()` in `cleanupSession()`.

### How to Confirm

```bash
journalctl --user -u msyncd -f | grep -E "(Background sync.*finished|removing activity|Invalid profile)"
```

If `"Invalid profile"` appears after `"Background sync finished"`, the reschedule
in `syncStatusChanged()` failed and the schedule was lost.

---

## Bug 5 (Minor, USE_IPHB Path Only): Inverted Logic in `removeAlarmEvent()`

**File:** `msyncd/SyncScheduler.cpp`, line 297

```cpp
if (err < false)  // bool < false is always false — dead code
```

This error-logging path can never execute. Not directly related to the backup issue
(this is the IPHB path, not USE_KEEPALIVE), but worth fixing.

---

## Diagnosis Plan

### Step 1: Collect Logs (48 hours)

```bash
# Capture full msyncd debug log for 48 hours to catch two backup cycles:
journalctl --user -u msyncd -f --no-pager | tee /tmp/msyncd-debug.log
```

### Step 2: Analyze Key Events

```bash
grep -n -E "(BackgroundSync|startScheduledSync|disabled period|Sync scheduled|next.*sync|Frequency|removing activity)" /tmp/msyncd-debug.log
```

### Step 3: Verify Bug 1 with Restart Workaround

After the first backup completes successfully:
```bash
systemctl restart --user msyncd
```

If the backup triggers successfully the next day with the restart but not without it,
Bug 1 (missing `stop()`) is confirmed as the primary cause.

### Step 4: Apply the Fix and Test

Apply the one-line fix to `BackgroundSync::set()` (add `stop()` before `wait()`)
and test over several days to verify scheduled backups trigger consistently.

---

## Priority Assessment

| Bug | Severity | Likely Cause of Reported Issue? |
|-----|----------|--------------------------------|
| 1 — Missing `stop()` | **Critical** | **Yes — primary suspect** |
| 2 — ±5 min window | Medium | Possible contributing factor |
| 3 — Boundary quantization | Low–Medium | May cause inconsistency |
| 4 — Double-scheduling race | Medium | Can cause permanent loss |
| 5 — Dead error logging | Low | No impact (wrong code path) |

Bug 1 alone fully explains the symptom "backup only works once after creation" and
should be fixed first. Bugs 2–4 may contribute to intermittent failures and should
be addressed as follow-up.

