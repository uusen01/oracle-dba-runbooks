# Invalid Objects Procedure

How to find, recompile, and diagnose INVALID database objects. A backlog of invalids is
common after deployments and patches; the goal is to clear the noise and distinguish
auto-healing invalids from genuine, broken dependencies.

> **Applies to:** Oracle 19c / 21c, single-instance and RAC, multitenant-aware.
> Placeholders throughout; nothing references any real or confidential environment.

---

## Background

Objects (packages, views, triggers, synonyms, procedures) become INVALID when something
they depend on changes — a deploy, a DDL change, or a patch. Oracle auto-recompiles an
invalid object on first use, but a backlog hides real compilation failures and adds
latency, so DBAs clear invalids proactively and investigate any that won't compile.

---

## 1. Find invalid objects

```sql
-- Count by owner/type (skip Oracle-maintained schemas)
SELECT owner, object_type, COUNT(*)
  FROM dba_objects
 WHERE status='INVALID'
   AND owner NOT IN ('SYS','SYSTEM','OUTLN','DBSNMP','APPQOSSYS','AUDSYS')
 GROUP BY owner, object_type ORDER BY 3 DESC;

-- Detail with last-DDL time (clusters of the same timestamp = one event, e.g. a deploy)
SELECT owner, object_type, object_name,
       TO_CHAR(last_ddl_time,'YYYY-MM-DD HH24:MI') AS last_ddl
  FROM dba_objects WHERE status='INVALID'
   AND owner NOT IN ('SYS','SYSTEM','OUTLN','DBSNMP','APPQOSSYS','AUDSYS')
 ORDER BY owner, object_type, object_name;
```
Record the count against your known baseline (kept in the Daily DBA Checklist log).

## 2. Recompile

**Whole schema** (review first):
```sql
EXEC DBMS_UTILITY.COMPILE_SCHEMA(schema => '<SCHEMA>', compile_all => FALSE);
```
**Database-wide** (the standard post-patch recompile):
```sql
@?/rdbms/admin/utlrp.sql
```
**A single object** (targeted):
```sql
ALTER PACKAGE <owner>.<name> COMPILE;
ALTER PACKAGE <owner>.<name> COMPILE BODY;
ALTER VIEW    <owner>.<name> COMPILE;
ALTER TRIGGER <owner>.<name> COMPILE;
```
(CDB: recompile within each affected PDB — `ALTER SESSION SET CONTAINER`, or run
`utlrp.sql` per container.)

## 3. Re-check and isolate the real failures

```sql
SELECT COUNT(*) FROM dba_objects WHERE status='INVALID'
 AND owner NOT IN ('SYS','SYSTEM','OUTLN','DBSNMP','APPQOSSYS','AUDSYS');
```
Anything **still invalid** after a recompile is a genuine break, not auto-heal noise.
Find out why:
```sql
SELECT owner, name, type, line, position, text
  FROM dba_errors
 WHERE owner = '<OWNER>' AND name = '<OBJECT>'
 ORDER BY sequence;
```
Typical causes: a referenced table/column was dropped or renamed, a dependent object is
also invalid (compile dependencies first), a missing grant/synonym, or a wrong build/patch
order.

## 4. Fix the dependency

- Recompile dependencies in order (a package body can't compile if its spec or a
  referenced object is invalid).
- Restore the missing/renamed object or correct the code, then recompile.
- If introduced by a deployment, coordinate with the release owner — it may be a bad or
  incomplete deploy that should be corrected at the source.

---

## After a patch

`utlrp.sql` is part of the patch flow. Always compare the post-patch invalid count to the
pre-patch baseline (see the **oracle-patching-runbooks** Post-Patch Validation). New
invalids that won't compile are a patch follow-up item, not "normal."

---

## Key principles

- **Recompile first, then judge** — many invalids clear immediately and were just stale.
- **A still-invalid object after recompile is a real problem** — read `DBA_ERRORS`.
- **Compile dependencies in order** — spec/base objects before bodies/dependents.
- **Track against a baseline** so you notice when a deploy introduces breakage.
