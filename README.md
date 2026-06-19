<p align="center">
  <img src="screenshots/oracle-dba-runbooks-banner.png" width="100%">
</p>

# Oracle DBA Runbooks

Operational **runbooks for day-to-day Oracle production support** — the daily routine,
startup/shutdown, health checks, and the incident-response procedures a Senior DBA reaches
for when something breaks. These are the "what do I do right now" guides that keep
production running and turn 2am pages into calm, repeatable actions.

Where the other repositories in this portfolio go deep on one topic (backups, tuning,
patching, multitenant), this one is the **on-call operations playbook**: broad, practical,
and incident-focused.

---

## Purpose

Production support is about doing the right thing, consistently, under time pressure.
These runbooks encode that: a daily checklist so nothing is missed, clean
startup/shutdown sequences, focused troubleshooting for the most common incidents
(connectivity, space, blocking, invalid objects), an emergency recovery checklist for when
production is down, and a blameless post-incident template so every incident makes the
system more resilient. Each is written to be followed step by step by any DBA on the team.

---

## Supported scope

| Dimension | Coverage |
|---|---|
| Oracle versions | **19c, 21c** (procedures generalize across current releases) |
| Architectures | Single-instance, RAC, multitenant (CDB/PDB) where relevant |
| Operating systems | Linux and Windows |
| Focus | Daily operations + **incident response** (distinct from project work) |

---

## ⚠️ Disclaimer — sanitized & generic by design

All runbooks and sample outputs are **fully sanitized and generic**. They contain **no**
real hostnames, IPs, SIDs, service names, credentials, or employer/company data. Database
and container names (`ORADEMO`, `SALES_PDB`, `RPT_PDB`), schema names, and all values in
`sample_outputs/` are **fictional**, written to show what a procedure looks like in
practice. These are reference procedures for portfolio and educational use — validate in a
non-production environment and apply under your own change-control process.

---

## Runbook index

| Runbook | Use it to… |
|---|---|
| [Daily DBA Checklist](runbooks/Daily%20DBA%20Checklist.md) | Run the start-of-day health and recoverability pass |
| [Startup and Shutdown Procedure](runbooks/Startup%20and%20Shutdown%20Procedure.md) | Stop/start DB + listener + PDBs cleanly (single-instance & RAC) |
| [Database Health Check Procedure](runbooks/Database%20Health%20Check%20Procedure.md) | Run a deeper weekly/after-change health assessment |
| [Listener Troubleshooting](runbooks/Listener%20Troubleshooting.md) | Resolve "can't connect" — ORA-12541/12514/12154 and more |
| [Space Management Procedure](runbooks/Space%20Management%20Procedure.md) | Prevent/resolve ORA-01653, ORA-00257, ORA-01652, undo issues |
| [Blocking Sessions Guide](runbooks/Blocking%20Sessions%20Guide.md) | Diagnose and clear lock contention ("the app is hung") |
| [Invalid Objects Procedure](runbooks/Invalid%20Objects%20Procedure.md) | Recompile invalids and isolate real dependency breaks |
| [Backup Verification Procedure](runbooks/Backup%20Verification%20Procedure.md) | Confirm backups ran, are present, and can actually restore |
| [Emergency Recovery Checklist](runbooks/Emergency%20Recovery%20Checklist.md) | Work a production-down incident safely and fast |
| [Post-Incident Review Template](runbooks/Post-Incident%20Review%20Template.md) | Document an incident blamelessly and drive prevention |

Sanitized, annotated example output for the key procedures is in
[`sample_outputs/`](sample_outputs/).

---

## How these fit a support day

```
  Morning            During the day (as needed)                 After an incident
  -------            --------------------------                 -----------------
  Daily DBA          Listener / Space / Blocking /              Emergency Recovery
  Checklist   --->   Invalid Objects troubleshooting   --->     Checklist (restore)
     |               Startup/Shutdown for changes                      |
     v                                                                 v
  Backup Verification   ---------------------------------->   Post-Incident Review
  (recoverability)                                            (prevent recurrence)
        ^                                                              |
        +----------- Database Health Check (weekly, deeper) <----------+
```

The sample outputs trace one consistent environment (`ORADEMO`) through a realistic
arc — clean daily check, a few common incidents, and an FRA-full outage that flows into
the post-incident review.

---

## Sample output

From the daily blocking-session check (fictional data):

```
 WAITER_SID WAITER_USER  EVENT                          SECONDS_IN_WAIT BLOCKER_SID BLOCKER_USER
----------- ------------ ------------------------------ --------------- ----------- ------------
        418 APP_USER     enq: TX - row lock contention              742         330 BATCH_USER
```

From the ORA-00257 emergency (fictional data):

```
/u01/app/oracle/fast_recovery_area          100.0     <-- FRA full, archiver stalled
RMAN> BACKUP ARCHIVELOG ALL;  RMAN> DELETE ARCHIVELOG ... BACKED UP ...;
-- FRA now 41% used; archiver resumed.
```

Each file in [`sample_outputs/`](sample_outputs/) ends with a **"Read:"** note explaining
the diagnosis and action.

---

## Operational Screenshots (Proof of Work)

Runbooks describe what *should* happen. This section shows the work actually being done — real terminal sessions from a sanitized demo database (`ORADEMO`), walking from symptom to diagnosis to verified fix. Every value is fictional; there are no real hostnames, SIDs, schemas, or company data. The point isn't the commands — it's the **judgment**: how an experienced operator reads the output, finds the root cause rather than the symptom, and confirms service is genuinely restored before calling it done.

---

### 1 · Clearing a blocking-session chain ("the app is hung")

![Blocking session chain on ORADEMO showing two APP_USER sessions waiting on a root blocker, killed and cleared](screenshots/blocking_sessions.png)

**Problem demonstrated.** The classic "the application froze" page. Two `APP_USER` sessions have been stuck for 12+ minutes on `enq: TX - row lock contention`, both pointing at the same blocker (session 330).

**What an experienced DBA concludes.** The instinct of a junior DBA is to kill the two sessions that are *complaining*. The senior move is the second query: confirm whether the blocker is itself blocked. Here 330 is `INACTIVE` with a 905-second `LAST_CALL_ET` — a batch job that errored mid-transaction and is now sitting on a row lock with no commit. That makes 330 the **root blocker**, and the two waiters are victims. Kill 330 (with the batch owner's sign-off) and the waiters clear instantly — which they do (`COUNT(*) = 0`).

**Troubleshooting takeaway.** Always walk the chain to the *root* blocker before killing anything; the loudest session is rarely the cause. And clearing the lock is only half the fix — the real follow-up is the batch job's commit/error handling so the same orphaned transaction can't recur.

---

### 2 · Recovering an archiver-stuck outage (ORA-00257)

![ORA-00257 diagnosis on ORADEMO showing a full Fast Recovery Area reclaimed via RMAN, restoring service](screenshots/emergency_archiver_stuck.png)

**Problem demonstrated.** A production-down incident: new connections are failing with `ORA-00257` ("Archiver error"). The database has stopped accepting work because it can't archive its redo logs.

**What an experienced DBA concludes.** The Fast Recovery Area is 100% full and `0.0%` is reclaimable — which is the real story. The space didn't just fill; *nothing was reclaimable* because the nightly archive-log backup had been failing silently for two nights, so no logs had been backed up and released. The fix is deliberately **not** deleting archive logs at the OS level (that would silently destroy recoverability). It's backing them up with RMAN first, then deleting the backed-up copies — FRA drops to 41%, the archiver resumes, and connections succeed. Total downtime ~33 minutes.

**Troubleshooting takeaway.** `ORA-00257` is a space symptom of a backup problem. Reclaim space *through* RMAN, never around it. The lasting fix lives in monitoring: alert on backup-job failures **and** on FRA percent-used, so the next one is caught as a warning at 80%, not an outage at 100%.

---

### 3 · A clean daily health pass (disciplined operations)

![Daily checklist run on ORADEMO showing instance OPEN, ARCHIVELOG, PDBs read-write, backups completed, zero issues](screenshots/daily_checklist.png)

**Problem demonstrated.** Not an incident — the *absence* of one. This is the start-of-day pass that prevents the other screenshots from ever happening.

**What an experienced DBA concludes.** Everything that matters is green in one glance: instance `OPEN` and in `ARCHIVELOG`, both application PDBs `READ WRITE`, last night's incremental `COMPLETED`, no failed jobs, no blocking, zero invalid objects, FRA at a comfortable 59%. There is nothing to action — and that is the point. The value of running the same pass every morning is that a *bad* line (a `FAILED` backup, a blocking chain, FRA at 95%) jumps out immediately against the baseline.

**Troubleshooting takeaway.** Consistency is the cheapest reliability tool you have. A two-minute daily checklist turns most 2am incidents into routine daytime maintenance, because the early warning was visible days before the failure.

---

### 4 · Listener triage done right (ORA-12514)

![Listener status on ORADEMO showing a missing service traced to a MOUNTED PDB, resolved by opening it and re-registering](screenshots/listener_status.png)

**Problem demonstrated.** Users can't connect to one service (`rpt_pdb`) with `ORA-12514` ("service not known"), while everything else works. The tempting wrong conclusion is "the listener is broken."

**What an experienced DBA concludes.** The listener is fine — 15 days uptime, other services `READY`. The tell is that `rpt_pdb` is **missing** from the Services Summary, and the reason is one query away: the PDB is `MOUNTED`, not `OPEN`, so its service never registered. Open the PDB, force registration, and the service comes back `READY`. The listener was never the problem; the closed PDB was.

**Troubleshooting takeaway.** `ORA-12514` is a *registration* problem, not a listener-down problem. Triage in order: confirm the listener is up, then confirm the database/PDB is open and the service is `READY`. Most "connection" incidents are resolved at the database layer, not the listener.

---

### 5 · Proving backups are recoverable, not just present

![Backup verification on ORADEMO showing completed backups, clean crosscheck, restore validate with no corruption, full coverage](screenshots/backup_verification.png)

**Problem demonstrated.** The question that ends careers when answered wrong: *"If we lost this database right now, could we get it back?"* A "backup succeeded" email does not answer it.

**What an experienced DBA concludes.** This is verification, not hope. Both backups `COMPLETED`; `CROSSCHECK` confirms all 66 pieces are physically `AVAILABLE`; `RESTORE DATABASE VALIDATE` proves the backups are actually usable with no corruption; `REPORT NEED BACKUP` returns no rows, so everything is protected inside the 14-day recovery window; and `v$database_block_corruption` is empty. That chain is the difference between *"we take backups"* and *"we have proven we can recover."*

**Troubleshooting takeaway.** A backup you have never restored is a theory, not a backup. Validate routinely — crosscheck, `RESTORE ... VALIDATE`, and a periodic real restore to a scratch host — so recoverability is a measured fact, not an assumption you discover is false during an outage.

---

> **All screenshots are fully sanitized and fictional.** Database/container names (`ORADEMO`, `SALES_PDB`, `RPT_PDB`), users, paths, and values are illustrative demo data created for this portfolio — no production, employer, or confidential information is shown. Each capture mirrors the annotated text in [`sample_outputs/`](sample_outputs/), where every example ends with a **"Read:"** note explaining the diagnosis and action.

---

## Repository structure

```
oracle-dba-runbooks/
├── README.md
├── LICENSE
├── .gitignore
├── runbooks/            # the 10 operational + incident-response procedures
├── sample_outputs/      # fictional, sanitized command output for the key procedures
└── screenshots/         # sanitized terminal captures — see "Operational Screenshots"
```

---

## Related repositories

These runbooks reference deeper, topic-specific repos in the same portfolio:

- **oracle-health-checks** — the read-only scripts behind the daily/health checks.
- **oracle-rman-scripts** — backup/recovery scripts and the full restore guide.
- **oracle-performance-tuning** — detailed diagnostics for slow databases.
- **oracle-patching-runbooks** — Release Update, RAC, Data Guard, and rollback procedures.
- **oracle-cdb-pdb-administration** — multitenant (CDB/PDB) administration.

---

## Core principles

- **Find and fix small problems early** — the daily checklist exists to prevent incidents.
- **Restore service first in an outage; preserve evidence; reconcile state before
  "all clear."**
- **Root cause, not symptom** — clear the blocker *and* fix the missing commit; reclaim
  space *and* fix the failed job.
- **Every incident is a lesson** — the blameless review turns a bad night into a more
  resilient system.

---

## Future enhancements

- **Severity & escalation matrix** (Sev 1–4 with response targets and contacts).
- **On-call handover template** for shift changes.
- **Per-procedure automation** linking each check to a runnable script.
- **SLO / uptime tracking** sheet to trend availability over time.

---

## License

Released under the [MIT License](LICENSE) — free to use, adapt, and share with
attribution.
