# Backup Verification Procedure

How to confirm — every day — that backups actually ran, succeeded, and are usable. A
backup you have not verified is not a backup. This is the single most important recurring
check a production DBA performs.

> **Applies to:** Oracle 19c / 21c using RMAN, single-instance and RAC. Placeholders
> throughout; nothing references any real or confidential environment. See the
> **oracle-rman-scripts** repo for the backup scripts themselves.

---

## 1. Did last night's backup run and succeed?

```sql
SELECT input_type, status,
       TO_CHAR(start_time,'YYYY-MM-DD HH24:MI') AS start_time,
       TO_CHAR(end_time,'YYYY-MM-DD HH24:MI')   AS end_time,
       time_taken_display AS duration
  FROM v$rman_backup_job_details
 WHERE start_time > SYSDATE - 1
 ORDER BY start_time DESC;
```
Look for `status = COMPLETED`. `COMPLETED WITH WARNINGS` or `FAILED` is a same-day action
item — do not assume "it usually works."

## 2. Is each backup type current?

```sql
SELECT object_type, TO_CHAR(MAX(completion_time),'YYYY-MM-DD HH24:MI') AS last_done
  FROM (
    SELECT 'Datafile' obj_marker, completion_time, 'Datafile' object_type FROM v$backup_set
  ) GROUP BY object_type;
-- Simpler per-type recency via RMAN:  RMAN> LIST BACKUP SUMMARY;
```
Confirm a recent **full/incremental**, **archive log**, and **control file** backup exist
within their expected schedule. A missing daily archive backup is an RPO and FRA risk.

## 3. Are the backups physically present? (crosscheck)

```
RMAN> CROSSCHECK BACKUP;
RMAN> CROSSCHECK ARCHIVELOG ALL;
RMAN> LIST EXPIRED BACKUP SUMMARY;     -- expect none
```
`EXPIRED` means RMAN's record exists but the file is gone from disk/tape — a real gap to
investigate and clean up.

## 4. Could the backups actually restore? (the real test)

```
RMAN> RESTORE DATABASE VALIDATE;       -- checks backups are usable, restores nothing
RMAN> RESTORE CONTROLFILE VALIDATE;
RMAN> BACKUP VALIDATE CHECK LOGICAL DATABASE;   -- block-level integrity
```
```sql
SELECT * FROM v$database_block_corruption;      -- expect no rows
```
This is what separates "we take backups" from "we can recover" — run validate regularly
(e.g., weekly), not just when desperate.

## 5. Periodic restore drill (the gold standard)

On a schedule (e.g., monthly/quarterly), **actually restore** the latest backup to a
scratch host or use RMAN `DUPLICATE`, and time it. The number you get is your real RTO,
and the drill proves the whole chain — backup, archive logs, control file, and procedure —
works end to end. (See the **oracle-rman-scripts** Restore and Recovery Guide.)

---

## FRA / retention sanity

```sql
SELECT name, ROUND(space_used*100/NULLIF(space_limit,0),1) AS pct_used
  FROM v$recovery_file_dest;
RMAN> REPORT NEED BACKUP;              -- files not protected within the retention window
RMAN> SHOW RETENTION POLICY;
```
`REPORT NEED BACKUP` returning rows means something is outside the recovery window — fix
it the same day.

---

## Daily verification checklist

- [ ] Last backup job `COMPLETED` (no FAILED / WARNINGS).
- [ ] Full, archivelog, and control file backups current to schedule.
- [ ] `CROSSCHECK` clean; no `EXPIRED` backups.
- [ ] FRA has headroom; `REPORT NEED BACKUP` returns nothing.
- [ ] (Weekly) `RESTORE ... VALIDATE` and `BACKUP VALIDATE CHECK LOGICAL` pass.
- [ ] (Periodic) restore drill completed within RTO and documented.

---

## Key principle

**Verification is part of the backup, not an optional extra.** The day you need to
recover is the wrong day to discover the backups have been silently failing. Check daily,
validate weekly, restore-drill periodically.
