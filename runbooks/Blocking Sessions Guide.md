# Blocking Sessions Guide

Diagnose and resolve lock contention — the "the application is hung / frozen" incident.
The method: confirm blocking exists, find the **root blocker**, understand why it's
holding the lock, and clear it with the least disruptive action.

> **Applies to:** Oracle 19c / 21c, single-instance and RAC. Placeholders throughout;
> nothing references any real or confidential environment.

---

## 1. Confirm and see the blocking chain

```sql
SELECT w.inst_id, w.sid AS waiter_sid, w.serial# AS waiter_ser,
       w.username AS waiter_user, w.event, w.seconds_in_wait,
       b.sid AS blocker_sid, b.serial# AS blocker_ser, b.username AS blocker_user
  FROM gv$session w
  JOIN gv$session b
    ON b.inst_id = w.blocking_instance AND b.sid = w.blocking_session
 WHERE w.blocking_session IS NOT NULL
 ORDER BY w.seconds_in_wait DESC;
```
No rows = no blocking. Otherwise you have the waiter→blocker pairs.

## 2. Identify the ROOT blocker

The root blocker **blocks others but is not itself blocked**. In a chain (A blocks B
blocks C), fixing A clears the whole chain. Watch for a blocker whose own
`blocking_session` is null.

## 3. Understand what's locked and why

```sql
-- What object and lock mode the blocker holds
SELECT s.sid, s.serial#, o.owner||'.'||o.object_name AS object, l.locked_mode
  FROM gv$locked_object l
  JOIN dba_objects o ON o.object_id = l.object_id
  JOIN gv$session  s ON s.sid = l.session_id AND s.inst_id = l.inst_id
 WHERE s.sid = &blocker_sid;

-- What the blocker is / was running, and whether it's idle in a transaction
SELECT sid, serial#, status, last_call_et AS secs_since_last_call, sql_id, prev_sql_id
  FROM gv$session WHERE sid = &blocker_sid;
```
Common causes: an uncommitted transaction (batch that errored mid-run), an application
holding a row lock while idle, or a long DML/DDL on a hot table.

## 4. Resolve — least disruptive first

1. **Engage the session owner.** The right fix is usually for the application/batch owner
   to commit or roll back. A row lock held by active, legitimate work should be allowed to
   finish.
2. **Wait it out** if the blocker is genuinely progressing (check `last_call_et`,
   `v$session_longops`) and the window allows.
3. **Kill the blocker** only with owner confirmation, when it is stuck/idle-in-transaction
   and impact is understood:
   ```sql
   ALTER SYSTEM KILL SESSION '<sid>,<serial#>,@<inst_id>' IMMEDIATE;
   ```
   (`KILL SESSION` rolls back the session's transaction — confirm the data impact first.)

## 5. Confirm resolution

Re-run the Section 1 query — the waiters should clear. Verify the application recovers.

---

## Special cases

- **Idle blocker in a transaction** (`status=INACTIVE`, holds a lock, `last_call_et`
  growing): classic application bug — opened a transaction and didn't commit. Kill with
  owner OK, then fix the app behavior.
- **`enq: TX - row lock contention`**: row-level lock; usually two sessions updating the
  same row. **`enq: TM`**: table-level (often a missing FK index causing full-table locks
  on child DML) — add the index to prevent recurrence.
- **Self-deadlock / ORA-00060**: Oracle auto-resolves deadlocks by killing one statement
  and writing a trace; investigate the app logic and access order.

---

## Prevention

- Application should commit promptly and keep transactions short.
- Index foreign keys to avoid TM-lock escalation on child-table DML.
- Monitor for long-held locks proactively (the **oracle-health-checks** repo's
  `blocking_sessions.sql`) so chains are caught before users call.

---

## Key principles

- **Find the root blocker** — don't kill waiters.
- **Engage the owner before killing** — a kill rolls back their transaction.
- **Fix the cause** (missing commit, missing FK index), not just the symptom, so the same
  hang doesn't return tomorrow.
