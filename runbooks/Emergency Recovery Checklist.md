# Emergency Recovery Checklist

The action checklist for a production-down database incident. The objective is to restore
service safely and quickly while preserving the evidence needed for root-cause analysis.
Work it calmly and in order — most outages have a known, recoverable cause.

> **Applies to:** Oracle 19c / 21c. Use during an active incident. Placeholders
> throughout; nothing references any real or confidential environment. Engage Oracle
> Support (Sev 1) in parallel for anything you can't quickly resolve.

---

## First 5 minutes — stabilize, don't worsen

- [ ] **Declare an incident.** Notify stakeholders and on-call; start a timestamped log.
- [ ] **Do not make blind changes.** Capture state first: the exact error, the alert log
      tail, and what changed recently (deploy, patch, storage, network).
- [ ] **Classify the symptom** — it dictates the path:

| Symptom | Likely cause | Go to |
|---|---|---|
| Instance down / won't start | Crash, parameter, memory, host issue | Section A |
| Up but no one can connect | Listener / service / PDB closed | Section B |
| Datafile/tablespace error | Lost or corrupt datafile | Section C |
| Whole DB lost / storage gone | Disaster | Section D |
| Hung / frozen | Blocking, latch, resource exhaustion | Section E |

---

## A. Instance down / won't start

```sql
sqlplus / as sysdba
SQL> STARTUP;
```
- Read the exact error. `STARTUP NOMOUNT` failing → spfile/memory/host. `MOUNT` failing →
  control file. `OPEN` failing → datafile/redo or needs recovery.
- Control file lost → restore it (see **oracle-rman-scripts** Restore Controlfile).
- After a crash, `STARTUP` performs instance recovery automatically — let it finish.
- (Windows) confirm the Oracle service started; (RAC) check `crsctl stat res -t`.

## B. Up but unreachable

- Listener down or service unregistered, or (CDB) the PDB is MOUNTED not OPEN.
- Follow **Listener Troubleshooting.md**: `lsnrctl status`, open the PDB
  (`ALTER PLUGGABLE DATABASE ALL OPEN;`), `ALTER SYSTEM REGISTER;`.

## C. Datafile / tablespace failure

```sql
SELECT file#, name, status FROM v$datafile WHERE status NOT IN ('ONLINE','SYSTEM');
SELECT * FROM v$recover_file;     -- files needing recovery
```
- Single datafile, DB can stay up:
  ```sql
  ALTER DATABASE DATAFILE <n> OFFLINE;   -- then RMAN restore+recover, then ONLINE
  ```
- Block corruption only: `RMAN> RECOVER CORRUPTION LIST;`
- Full restore/recover sequence is in the **oracle-rman-scripts** Restore and Recovery
  Guide.

## D. Disaster (whole database lost)

- Follow the full restore path: restore SPFILE → control file → database → recover →
  `OPEN RESETLOGS` (**oracle-rman-scripts** Restore and Recovery Guide), or
- **Fail over to the standby** if Data Guard exists (fastest path — activate/switch the
  standby to primary), then re-establish protection afterward.

## E. Hung / frozen

- Blocking chain → **Blocking Sessions Guide.md** (clear the root blocker).
- FRA full / archiver stuck (ORA-00257) → **Space Management Procedure.md** (reclaim via
  RMAN).
- Sessions/processes maxed (ORA-00020) → identify the source (often a pool leak); raise
  limits only if real growth.

---

## Before declaring "all clear"

- [ ] Database OPEN, PDBs OPEN, listener up, services registered.
- [ ] No ongoing `ORA-` errors in the alert log; no block corruption.
- [ ] If you flashed back / restored / failed over, **binary, dictionary, and data states
      are consistent**, and you've taken a **fresh full backup**.
- [ ] Application owner confirms functional recovery.

## After the incident

- [ ] Keep all logs and evidence for the Oracle SR and the review.
- [ ] Complete the **Post-Incident Review Template.md** (blameless) — what failed, why,
      timeline, real RTO, and the fix to prevent recurrence.

---

## Key principles

- **Restore service first; reconcile and tidy second** — but never leave inconsistent
  binary/dictionary/data state once stable.
- **Preserve evidence** as you go; you'll need it for RCA and Support.
- **Lean on what you prepared** — tested backups, a standby, and these runbooks are what
  make a fast, safe recovery possible.
