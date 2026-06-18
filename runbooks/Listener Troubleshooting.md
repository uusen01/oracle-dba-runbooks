# Listener Troubleshooting

Diagnose and resolve Oracle Net connectivity problems — the "I can't connect to the
database" calls. The listener is the front door; most connectivity incidents are listener,
service-registration, or TNS issues rather than the database itself.

> **Applies to:** Oracle 19c / 21c, single-instance and RAC, Windows and Linux.
> Placeholders throughout; nothing references any real or confidential environment.

---

## Triage by error (start here)

| Error the user reports | Most likely cause | Go to |
|---|---|---|
| `ORA-12541: TNS:no listener` | Listener is down | Section 1 |
| `ORA-12514: ... service not known` | Service not registered with listener | Section 2 |
| `ORA-12505: ... SID not known` | Wrong SID / using SID instead of service | Section 2 |
| `ORA-12154: TNS:could not resolve` | Client-side TNS name/alias problem | Section 3 |
| `ORA-12520 / ORA-12516` | No handler / connection storm / limits | Section 4 |
| Connects sometimes, not others (RAC) | Service running on a subset of nodes | Section 5 |

---

## 1. Is the listener up?

```
lsnrctl status <listener_name>
```
- If it's not running, start it: `lsnrctl start <listener_name>`.
- Confirm it's listening on the expected port/host (check the `Listening Endpoints`).
- (Windows) verify the listener **service** is running.

## 2. Is the database service registered?

`lsnrctl status` lists services. If the app's service is missing or shows
`status UNKNOWN/BLOCKED`:

- Confirm the database (and PDB, if multitenant) is **OPEN** — a closed PDB does not
  register its service.
- Check/force service registration from the instance:
  ```sql
  SHOW PARAMETER local_listener;          -- should point to the listener
  ALTER SYSTEM REGISTER;                   -- force immediate re-registration
  ```
- For a PDB service, confirm it exists and is started:
  ```sql
  ALTER SESSION SET CONTAINER = <pdb>;
  EXEC DBMS_SERVICE.START_SERVICE('<service>');
  ```
- Connect using the **service name** (not SID) for multitenant:
  `sqlplus user@//host:1521/<service_name>`

## 3. Client-side resolution (ORA-12154)

- Verify the alias in the client `tnsnames.ora` matches what the app uses, and
  `TNS_ADMIN` points to the right directory.
- Test resolution and reachability:
  ```
  tnsping <alias>
  ```
- Confirm host/port are reachable (firewall, DNS) from the client.

## 4. No handler / connection limits (ORA-12516 / ORA-12520)

- Often a connection storm or hitting `PROCESSES`/`SESSIONS`:
  ```sql
  SELECT resource_name, current_utilization, limit_value
    FROM v$resource_limit WHERE resource_name IN ('processes','sessions');
  ```
- Check for a leaking connection pool (many INACTIVE sessions from one program).
- Raise limits only if concurrency legitimately grew (requires restart for `processes`).

## 5. RAC connectivity

```
srvctl status service -d <db>            # which nodes the service runs on
srvctl status listener
lsnrctl services
```
- Ensure the service is started on the intended instances:
  `srvctl start service -d <db> -s <service>`.
- Confirm SCAN listeners are up and SCAN name resolves to the SCAN VIPs.

---

## Useful diagnostics

```
lsnrctl status <listener>      # state, endpoints, services, uptime
lsnrctl services <listener>    # handlers and "ready/blocked" per service
tnsping <alias>                # client resolution + round-trip
```
The listener log (`$ORACLE_BASE/diag/tnslsnr/.../alert/log.xml`) records registration and
connection errors — check it when the cause isn't obvious.

---

## Resolution checklist

- [ ] Listener running and on the expected endpoint.
- [ ] Database/PDB OPEN and the service registered (`lsnrctl status` shows it READY).
- [ ] Client uses the correct service name and TNS alias.
- [ ] (RAC) service started on the right nodes; SCAN healthy.
- [ ] After fixing, the user reconnects successfully — confirm before closing.

Most listener incidents resolve at Section 1 or 2: the listener was down, or a
database/PDB wasn't open so its service never registered.
