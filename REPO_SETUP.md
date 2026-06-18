# Repository Setup Guide (publishing checklist)

> For you, Usen — NOT portfolio content. After you set the GitHub topics and confirm
> everything looks right, delete it (or keep it private). It explains where every file
> belongs, the topics to add, the screenshots to capture, and how to publish
> `oracle-dba-runbooks`.

---

## 1. Where every file belongs

```
oracle-dba-runbooks/                          <- repository root
├── README.md                                 <- main landing page
├── LICENSE                                    <- MIT license (GitHub auto-detects at root)
├── .gitignore                                <- keeps logs, secrets, real incidents out
├── REPO_SETUP.md                              <- THIS file (delete before/after publishing)
│
├── runbooks/                                  <- the 10 procedures
│   ├── Daily DBA Checklist.md
│   ├── Startup and Shutdown Procedure.md
│   ├── Database Health Check Procedure.md
│   ├── Listener Troubleshooting.md
│   ├── Space Management Procedure.md
│   ├── Blocking Sessions Guide.md
│   ├── Invalid Objects Procedure.md
│   ├── Backup Verification Procedure.md
│   ├── Emergency Recovery Checklist.md
│   └── Post-Incident Review Template.md
│
├── sample_outputs/                            <- fictional, sanitized command output
│   ├── daily_checklist_run_sample.txt
│   ├── startup_shutdown_sample.txt
│   ├── listener_status_sample.txt
│   ├── space_management_sample.txt
│   ├── blocking_sessions_sample.txt
│   ├── invalid_objects_sample.txt
│   ├── backup_verification_sample.txt
│   └── emergency_archiver_stuck_sample.txt
│
└── screenshots/
    └── README.md
```

Rationale: `README.md` and `LICENSE` at the root for GitHub; the procedures under
`runbooks/` (they ARE the deliverable and render as markdown); fictional command output
under `sample_outputs/`.

> The README requested a `docs` folder; for a runbook repository the procedures ARE the
> docs, so they live in `runbooks/` (clearer than a generic `docs/`). If you'd rather have
> a `docs/` folder, rename `runbooks/` — the README links would need updating to match.

---

## 2. GitHub topics (add via the ⚙️ next to "About")

```
oracle
oracle-database
dba
database-administration
production-support
runbook
incident-response
troubleshooting
oracle-19c
oracle-21c
on-call
sre
database-engineering
operations
```

Suggested **About** description:
> *Operational runbooks for Oracle production support (19c/21c) — daily checks,
> startup/shutdown, listener, space, blocking, backups, and incident response. Sanitized.*

---

## 3. Screenshots to capture (optional, high-impact)

From a lab DB, sanitized, dark terminal. Priority order:

1. `blocking_sessions.png` — blocker→waiter chain, root blocker identified
2. `emergency_archiver_stuck.png` — ORA-00257 diagnosis + RMAN reclaim
3. `daily_checklist.png` — a clean morning pass
4. `listener_status.png` — services READY (or a 12514 fix)
5. `backup_verification.png` — RESTORE...VALIDATE passing

See `screenshots/README.md`. The blocking + emergency pair demonstrates incident response.

---

## 4. Publishing steps

```bash
cd oracle-dba-runbooks
git init
git add .
git commit -m "Oracle DBA operational + incident-response runbooks (19c/21c)"
git branch -M main
git remote add origin https://github.com/uusen01/oracle-dba-runbooks.git
git push -u origin main
```

Then on github.com/uusen01/oracle-dba-runbooks:
1. Add the **topics** from section 2 and set the **About** description.
2. Confirm `LICENSE` shows "MIT" in the sidebar.
3. (Optional) add screenshots and link them in the README.
4. **Pin** the repo on your profile.
5. Ensure the profile README "Featured" link points here as a clickable markdown link.

---

## 5. Final sanitization pass before pushing

- [ ] No real hostnames, IPs, SIDs, service names, or DBIDs (placeholders only;
      `ORADEMO`, `SALES_PDB`, `RPT_PDB` and sample values are fictional).
- [ ] No credentials; no `tnsnames.ora`/`login.sql`/password files committed (.gitignore).
- [ ] No real completed incident reviews committed (`.gitignore` excludes `incidents/`,
      `INC-*.md`) — the Post-Incident Review is a blank TEMPLATE only.
- [ ] No USPS / AT&T / GM / employer identifiers in any file.
- [ ] Sample outputs are clearly fictional and labeled in the README disclaimer.
- [ ] Delete this `REPO_SETUP.md` if you don't want it public.

---

## Note — this is your "production support DBA" showcase

This repo positions you squarely as a production-support / on-call DBA — exactly the
framing many $135k–$170k Senior DBA roles use. It is distinct from
`oracle-patching-runbooks` (project/maintenance work): this one is daily operations +
incident response. The keep-scope-distinct point from the GitHub Portfolio Review is
satisfied — patching runbooks = planned change; these = run-the-business + firefighting.
The blocking-session and ORA-00257 stories also map to real troubleshooting you've done.
