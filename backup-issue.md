# Issue with backup triggering on Sailfish OS

Scheduled backup is not working sometimes. It is discussed on the forum: https://forum.sailfishos.org/t/automatic-daily-backup-only-works-once-after-creation/15563/14
Resposibility of backup scheduling and execution is on the `/usr/bin/msyncd` daemon,
it is part of the `buteo-syncfw-qt5-msyncd` package.

## Technical details

Package version in Sailfish OS 5.0.0.77 (recent stable release at 04/2026) is `0.11.6-1.24.12.jolla`

## Howto

Trigger the backup manually:
```bash
busctl call --user com.meego.msyncd /synchronizer com.meego.msyncd startSync s "nextcloud.Backup-28"
```
Daemon status:
```bash
systemctl status --user msyncd
```
Daemon restart:
```bash
systemctl restart --user msyncd
```

## Analysis

### Step 1: possible cause

    Disclaimer: following analysis was done by the **Claude Opus 4.6** model 

[possible cause analysis](backup-issue-analysis-step01.md)

### Step 2: journal log evidence (2026-05-05)

    Disclaimer: following analysis was done by the **Claude Opus 4.6** model

Manually triggered backup at 23:38, scheduled automatic backup at 00:16 did not execute.

[journal log analysis](backup-issue-analysis-step02.md)

**Key finding:** Bug 2 (±5 min validation window) is the **direct cause** — the timer fired
at 00:32 (16 min late) and was rejected. Bug 1 (missing `stop()`) is a **contributing factor**
that prevented the timer from being properly reset after the manual backup completed.

