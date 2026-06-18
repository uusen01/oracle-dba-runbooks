# Startup and Shutdown Procedure

The correct, ordered way to stop and start an Oracle database and its supporting services
— single-instance, multitenant, and RAC — so shutdowns are clean and startups leave the
database fully open and reachable.

> **Applies to:** Oracle 19c / 21c, Windows and Linux. Placeholders throughout; nothing
> references any real or confidential environment.

---

## Shutdown modes (know which to use)

| Mode | Behavior | When |
|---|---|---|
| `SHUTDOWN IMMEDIATE` | Rolls back active txns, disconnects sessions, clean | **Default** for planned shutdown |
| `SHUTDOWN TRANSACTIONAL` | Waits for active txns to finish, then closes | Minimize work loss, time permitting |
| `SHUTDOWN NORMAL` | Waits for all sessions to disconnect | Rarely — can hang on idle sessions |
| `SHUTDOWN ABORT` | Instant, no clean close; needs crash recovery on start | Emergencies / hung instance only |

Prefer `IMMEDIATE`. Use `ABORT` only when necessary; the next startup performs instance
recovery automatically.

---

## Planned shutdown (single instance)

1. Notify stakeholders; confirm the window.
2. Stop application connections / app tier first if possible.
3. Stop the listener so no new connections arrive:
   ```
   lsnrctl stop <listener_name>
   ```
4. Shut the database cleanly:
   ```sql
   sqlplus / as sysdba
   SQL> SHUTDOWN IMMEDIATE;
   ```
5. (Windows) stop the Oracle service if the host is being patched/rebooted:
   ```
   net stop OracleService<SID>
   ```

## Planned startup (single instance)

1. (Windows) start the service: `net start OracleService<SID>`.
2. Open the database:
   ```sql
   sqlplus / as sysdba
   SQL> STARTUP;
   ```
   Startup stages if needed: `STARTUP NOMOUNT` → `ALTER DATABASE MOUNT;` →
   `ALTER DATABASE OPEN;` (used during recovery scenarios).
3. Start the listener:
   ```
   lsnrctl start <listener_name>
   ```
4. **(CDB) open the PDBs** — they default to MOUNTED:
   ```sql
   ALTER PLUGGABLE DATABASE ALL OPEN;
   ```
   If they were not configured to auto-open, save state so next time they do:
   `ALTER PLUGGABLE DATABASE ALL SAVE STATE;`
5. Verify: instance OPEN, PDBs OPEN, services registered (see Daily DBA Checklist.md).

---

## RAC shutdown / startup

- **Stop one instance** (keep the database available on others):
  ```
  srvctl stop instance -d <db> -i <instance> -o immediate
  ```
- **Stop the whole database:** `srvctl stop database -d <db> -o immediate`
- **Start the database (all instances):** `srvctl start database -d <db>`
- **Stop/Start the full clusterware stack on a node (e.g., for OS work):**
  ```
  crsctl stop crs        # as root
  crsctl start crs       # as root
  ```
Use `srvctl` (not SQL\*Plus) for RAC instance/database lifecycle so Grid Infrastructure
stays in sync; check `crsctl stat res -t` after.

---

## Multitenant notes

- Opening the CDB does **not** open the PDBs — always include the PDB open step.
- Saved state (`SAVE STATE`) makes PDBs auto-open on future restarts; this is the fix for
  the recurring "PDB mounted but closed after reboot" outage.

---

## Common pitfalls

- **Forgetting the PDBs** — CDB is up, apps still fail because their PDB is MOUNTED.
- **Listener left down** — database is open but unreachable; always restart the listener.
- **`SHUTDOWN IMMEDIATE` appears to hang** — usually a large transaction rolling back; let
  it finish. Resort to `ABORT` only if truly stuck (recovery will run on startup).
- **RAC managed via SQL\*Plus** — bypasses clusterware; use `srvctl`.

---

## Verification after any startup

Instance OPEN · PDBs OPEN (CDB) · listener up and services registered · no `ORA-` errors
in the alert log during startup · application connectivity confirmed.
