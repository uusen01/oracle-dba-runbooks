# Screenshots

Optional terminal captures used in the main README and for portfolio presentation.
Images are not required for the runbooks to be useful — they exist to show recruiters and
hiring managers real production-support work at a glance.

## Capture guidance

- Run the commands against a **non-production / lab** database and capture the terminal
  output (dark theme, monospaced font, wide enough that output doesn't wrap).
- **Sanitize before capturing.** Confirm no real hostnames, IPs, service names,
  schema/user names, or company identifiers appear. Use a demo database (`ORADEMO`,
  `SALES_PDB`, `RPT_PDB`) so visible names are generic, as in `sample_outputs/`.
- Save as PNG with a descriptive name, e.g. `daily_checklist.png`, `blocking_sessions.png`.

## Recommended screenshots (highest portfolio signal first)

1. **`blocking_sessions.png`** — a blocker→waiter chain with the root blocker identified.
   The most recognizable real-incident image to any DBA reviewer.
2. **`emergency_archiver_stuck.png`** — ORA-00257 diagnosis + RMAN reclaim. Shows
   incident-response judgment (and that you don't delete files at the OS level).
3. **`daily_checklist.png`** — a clean morning pass. Signals disciplined operations.
4. **`listener_status.png`** — `lsnrctl status` with services READY (or a 12514 fix).
5. **`backup_verification.png`** — RESTORE...VALIDATE proving backups are usable.

Five well-chosen, sanitized screenshots beat capturing everything. The blocking-session
and emergency-archiver images together demonstrate incident-response capability, which is
exactly what production-support roles screen for.
