# Space Management Procedure

How to monitor and resolve space pressure across tablespaces, the Fast Recovery Area
(FRA), TEMP, and undo — preventing the space-related errors (ORA-01653, ORA-00257,
ORA-01652, ORA-30036) that take applications down.

> **Applies to:** Oracle 19c / 21c, single-instance and RAC, multitenant-aware.
> Placeholders throughout; nothing references any real or confidential environment.

---

## 1. Tablespace space (ORA-01653 / ORA-01654)

**Monitor:**
```sql
SELECT tablespace_name, ROUND(used_percent,1) AS pct_used_of_max
  FROM dba_tablespace_usage_metrics ORDER BY used_percent DESC;
```
(In a CDB, check per container — see the **oracle-cdb-pdb-administration** repo's
`pdb_tablespaces.sql`.)

**Resolve** when a tablespace nears its autoextend ceiling:
```sql
-- Add a datafile:
ALTER TABLESPACE <ts> ADD DATAFILE '/u01/.../ts_NN.dbf'
  SIZE 10G AUTOEXTEND ON NEXT 1G MAXSIZE 64G;
-- Or raise the ceiling on an existing file:
ALTER DATABASE DATAFILE '/u01/.../ts_01.dbf' AUTOEXTEND ON MAXSIZE 96G;
```
Then investigate *why* it grew (a load that didn't purge, missing archiving, unexpected
volume) — adding space without understanding growth just delays the next alert.

## 2. Fast Recovery Area (ORA-00257 archiver stuck)

**Monitor:**
```sql
SELECT name, ROUND(space_used*100/NULLIF(space_limit,0),1) AS pct_used
  FROM v$recovery_file_dest;
SELECT file_type, percent_space_used, percent_space_reclaimable
  FROM v$flash_recovery_area_usage;
```
**Resolve** a filling FRA by reclaiming through RMAN (never delete files at the OS level):
```
RMAN> BACKUP ARCHIVELOG ALL;
RMAN> DELETE ARCHIVELOG ALL BACKED UP 1 TIMES TO DISK COMPLETED BEFORE 'SYSDATE-1';
RMAN> DELETE OBSOLETE;
```
As a controlled stopgap, increase `DB_RECOVERY_FILE_DEST_SIZE` — alongside fixing the
backup that should have been reclaiming space.

## 3. TEMP (ORA-01652 unable to extend temp segment)

**Monitor:**
```sql
SELECT tablespace_name, ROUND(SUM(bytes_used)/1024/1024/1024,2) AS used_gb,
       ROUND(SUM(bytes_free)/1024/1024/1024,2) AS free_gb
  FROM v$temp_space_header GROUP BY tablespace_name;
```
**Resolve:** the durable fix is usually tuning the SQL doing a huge sort/hash (missing
index). If the workload genuinely needs more:
```sql
ALTER TABLESPACE TEMP ADD TEMPFILE '/u01/.../temp02.dbf'
  SIZE 8G AUTOEXTEND ON NEXT 1G MAXSIZE 32G;
```

## 4. Undo (ORA-30036 / ORA-01555)

**Monitor:** undo by status, and retention vs the longest query:
```sql
SELECT tablespace_name, status, ROUND(SUM(bytes)/1024/1024/1024,2) gb
  FROM dba_undo_extents GROUP BY tablespace_name, status;
SELECT MAX(maxquerylen) AS longest_query_s, MAX(tuned_undoretention) AS tuned_s
  FROM v$undostat WHERE begin_time > SYSDATE-1;
```
**Resolve:** size undo (or raise `UNDO_RETENTION`) to exceed the longest query; have large
batch jobs commit in chunks rather than holding huge active undo.

## 5. Segment / filesystem level

- Find the biggest growth drivers (`dba_segments`) — large fact/history tables are
  candidates for **partitioning** (enables pruning and fast purges) or archiving.
- Watch OS filesystems / ASM disk groups; a full mount point fails datafile autoextend
  even when the tablespace setting allows growth.

---

## Proactive practices

- **Alert thresholds:** monitor at ~85% of max and act before 100%.
- **Trend, don't react:** capture usage over time so you add space on a schedule, not in
  an incident. (The **oracle-health-checks** repo's scripts feed this.)
- **Purge/partition strategy:** for ever-growing tables, design retention/partitioning so
  space is reclaimed routinely.
- **Never** delete Oracle-managed files (archive logs, backups) at the OS level — always
  via RMAN, to preserve recoverability and repository consistency.

---

## Quick reference: error → action

| Error | Meaning | First action |
|---|---|---|
| ORA-01653/01654 | Tablespace can't extend | Add datafile / raise MAXSIZE |
| ORA-00257 | Archiver stuck, FRA full | RMAN backup+delete archivelogs; grow FRA |
| ORA-01652 | TEMP exhausted | Tune the big sort; add tempfile |
| ORA-30036 | Undo can't extend | Grow undo; chunk large transactions |
| ORA-01555 | Snapshot too old | Raise undo retention; tune long query |
