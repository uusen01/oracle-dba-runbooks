# Daily DBA Checklist

The start-of-day routine a production support DBA runs across each managed database to
confirm it is up, recoverable, and healthy before the business day ramps up. The goal is
to find and fix small problems while they are still small.

> **Applies to:** Oracle 19c / 21c, single-instance and RAC, Windows and Linux. All
> names, paths, and values are **placeholders / fictional**; nothing references any real
> or confidential environment.

---

## How to use this

Run once each morning, per database, before peak load. Record results in a simple log
(date, who, anything flagged, action taken) — the log is what turns reactive firefighting
into proactive trend management. Anything flagged becomes the day's work queue.

---

## The checklist

### 1. Availability
- [ ] Instance OPEN, correct open mode, ARCHIVELOG on:
      ```sql
      SELECT status FROM v$instance;
      SELECT open_mode, log_mode FROM v$database;
      ```
- [ ] (CDB) all PDBs OPEN, none unexpectedly RESTRICTED:
      ```sql
      SELECT name, open_mode, restricted FROM v$pdbs;
      ```
- [ ] Listener up and services registered (`lsnrctl status <listener>`).
- [ ] (RAC) all instances/resources ONLINE (`crsctl stat res -t`).

### 2. Backups & recoverability
- [ ] Last night's backup **completed** (see Backup Verification Procedure.md):
      ```sql
      SELECT input_type, status, end_time
        FROM v$rman_backup_job_details
       WHERE start_time > SYSDATE - 1 ORDER BY start_time DESC;
      ```
- [ ] Archiving healthy; FRA not near full (the ORA-00257 risk).

### 3. Space
- [ ] No tablespace near its autoextend ceiling (see Space Management Procedure.md).
- [ ] FRA and key filesystems/ASM disk groups have headroom.

### 4. Errors & alert log
- [ ] Scan the alert log since yesterday for `ORA-` errors, especially `ORA-00600`,
      `ORA-07445`, `ORA-01555`, block-corruption messages.
- [ ] No new corruption: `SELECT * FROM v$database_block_corruption;` (expect no rows).

### 5. Jobs & maintenance
- [ ] Scheduled jobs (stats, purges, backups, ETL) ran — no failures:
      ```sql
      SELECT owner, job_name, status, actual_start_date
        FROM dba_scheduler_job_run_details
       WHERE status='FAILED' AND actual_start_date > SYSDATE-1;
      ```

### 6. Sessions & contention
- [ ] No long-standing blocking chains (see Blocking Sessions Guide.md):
      ```sql
      SELECT blocking_session, sid, serial#, seconds_in_wait
        FROM v$session WHERE blocking_session IS NOT NULL;
      ```
- [ ] Session/process count comfortably below limits.

### 7. Objects
- [ ] Invalid objects at/below the known baseline (see Invalid Objects Procedure.md).

### 8. Performance sanity
- [ ] No abnormal load vs a normal morning; spot-check a representative query.

---

## Escalation

If any availability or recoverability item fails and you cannot resolve it quickly,
treat it as an incident: notify stakeholders, open a record, and work it (see Emergency
Recovery Checklist.md) rather than continuing the routine. Capture evidence as you go for
the Post-Incident Review.

---

## Tip

Automate the read-only collection (a single spooled script that runs the queries above) so
the daily pass takes minutes and the output is consistent and loggable. The companion
**oracle-health-checks** repository provides scripts for most of these checks.
