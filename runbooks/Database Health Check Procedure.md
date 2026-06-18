# Database Health Check Procedure

A deeper, structured health assessment beyond the daily glance — used weekly, after a
change, or when onboarding an unfamiliar database. It produces a documented snapshot of
availability, recoverability, space, performance, and configuration.

> **Applies to:** Oracle 19c / 21c, single-instance and RAC. Placeholders throughout;
> nothing references any real or confidential environment.

---

## When to run

- Weekly, as a scheduled deeper check than the daily pass.
- After any significant change (patch, parameter change, schema deploy).
- When taking over a database you don't know — this becomes your baseline.

---

## 1. Availability & configuration

```sql
SELECT instance_name, status, version, startup_time FROM v$instance;
SELECT name, open_mode, log_mode, force_logging, database_role FROM v$database;
SELECT name, value FROM v$parameter
 WHERE name IN ('sga_target','pga_aggregate_target','processes','sessions',
                'db_recovery_file_dest_size','undo_tablespace');
```
Record version, role, mode, and key parameters as the configuration baseline.

## 2. Recoverability

```sql
-- Last backups by type and any failures
SELECT input_type, status, start_time, end_time
  FROM v$rman_backup_job_details WHERE start_time > SYSDATE-14 ORDER BY start_time DESC;
-- Archiving / FRA headroom
SELECT * FROM v$recovery_file_dest;
```
Confirm a recent successful full backup and that the FRA is not filling. (See Backup
Verification Procedure.md.)

## 3. Space

```sql
-- Tablespaces near their autoextend ceiling
SELECT tablespace_name, ROUND(used_percent,1) AS pct_used_of_max
  FROM dba_tablespace_usage_metrics ORDER BY used_percent DESC;
-- Temp and undo headroom
SELECT tablespace_name, SUM(bytes)/1024/1024/1024 gb
  FROM dba_temp_files GROUP BY tablespace_name;
```
Flag anything above ~85% of max. (See Space Management Procedure.md.)

## 4. Performance profile

```sql
-- Wait-class profile (where time goes)
SELECT wait_class, ROUND(time_waited/100,1) AS time_s
  FROM v$system_wait_class WHERE wait_class <> 'Idle' ORDER BY time_waited DESC;
-- Heaviest cached SQL by elapsed
SELECT sql_id, ROUND(elapsed_time/1e6,1) AS elapsed_s, executions
  FROM v$sqlarea ORDER BY elapsed_time DESC FETCH FIRST 10 ROWS ONLY;
```
Note the dominant wait class and the top SQL. (The **oracle-performance-tuning** repo has
detailed scripts.)

## 5. Objects & integrity

```sql
SELECT owner, COUNT(*) FROM dba_objects WHERE status='INVALID'
 GROUP BY owner ORDER BY 2 DESC;
SELECT * FROM v$database_block_corruption;          -- expect no rows
SELECT comp_name, version, status FROM dba_registry; -- components VALID
```

## 6. Sessions & contention

```sql
SELECT status, COUNT(*) FROM v$session WHERE type='USER' GROUP BY status;
SELECT blocking_session, sid, serial#, seconds_in_wait
  FROM v$session WHERE blocking_session IS NOT NULL;
```

## 7. RAC / Data Guard (if applicable)

```sql
-- RAC: crsctl stat res -t  (all ONLINE; services on expected nodes)
-- Data Guard: apply lag / transport lag, zero gap
SELECT name, value FROM v$dataguard_stats WHERE name LIKE '%lag%';
```

---

## Output: the health report

Summarize findings in a short report with a status per area (Green / Yellow / Red),
the evidence, and any recommended actions with priority. Keep it — comparing this week's
report to last week's is how you spot slow-moving problems (growth, creeping invalids,
rising waits) before they become incidents.

| Area | Status | Notes / action |
|---|---|---|
| Availability | | |
| Recoverability | | |
| Space | | |
| Performance | | |
| Objects/Integrity | | |
| Sessions/Contention | | |

See **Daily DBA Checklist.md** for the lighter daily version and the topic runbooks for
fixes.
