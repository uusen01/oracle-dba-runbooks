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

## Repository structure

```
oracle-dba-runbooks/
├── README.md
├── LICENSE
├── .gitignore
├── runbooks/            # the 10 operational + incident-response procedures
├── sample_outputs/      # fictional, sanitized command output for the key procedures
└── screenshots/         # (optional) terminal captures for the portfolio
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
