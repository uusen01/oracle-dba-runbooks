# Post-Incident Review Template

A blameless template for documenting a database incident after service is restored. The
purpose is learning and prevention, not blame — focus on what the system and process
allowed to happen, and what to change so it doesn't recur.

> **Applies to:** any production database incident. Fill in per incident; keep completed
> reviews as a searchable knowledge base. Placeholders/fictional values below.

---

## Incident summary

| Field | Value |
|---|---|
| Incident ID / ticket | INC-XXXXX |
| Date / time detected | 2026-06-17 14:05 |
| Date / time resolved | 2026-06-17 14:38 |
| Duration (downtime) | 33 minutes |
| Severity | Sev 1 / 2 / 3 |
| Database / environment | ORADEMO (prod) |
| Detected by | Monitoring alert / user report / DBA check |
| Author | <name> |

**One-paragraph summary:** What happened, the customer/business impact, and how it was
resolved — in plain language a non-DBA can follow.

---

## Impact

- **Who/what was affected:** applications, users, batch, reporting, etc.
- **Business impact:** transactions delayed, SLA breach, data affected (and whether any
  data was lost — state clearly).
- **Scope:** one PDB / whole instance / cluster / site.

---

## Timeline (facts, with timestamps)

| Time | Event / action | Who |
|---|---|---|
| 14:05 | Alert: archiver stuck (ORA-00257); new connections failing | Monitoring |
| 14:08 | DBA acknowledged; incident declared | <name> |
| 14:12 | Identified FRA at 100%; nightly archive backup had failed 2 nights | <name> |
| 14:20 | RMAN backup of archivelogs + delete to reclaim FRA | <name> |
| 14:32 | Archiver resumed; connections restored | <name> |
| 14:38 | Verified open, services up, app confirmed; incident closed | <name> |

Keep this strictly factual and time-ordered — it is the backbone of the analysis.

---

## Root cause analysis

- **Trigger:** the immediate event (e.g., FRA filled and the archiver stalled).
- **Root cause:** the underlying reason it was *able* to happen (e.g., the archive-log
  backup job had been failing silently for two nights and no alert fired on job failure).
- **Contributing factors:** monitoring gaps, undersized FRA, missing alert, recent change.
- Use "5 whys" or a simple cause chain — push past the symptom to the systemic cause.

---

## Resolution

- What action restored service, and why it was chosen over alternatives.
- Whether any rollback/restore/flashback/failover was used, and the resulting state
  (confirm consistency and that a fresh backup was taken if applicable).

---

## What went well / what didn't

**Well:** fast detection, clear runbook, good teamwork, etc.
**Didn't:** silent job failure, no FRA alert, runbook step unclear, access delay, etc.
Be honest and specific — this is where the value is.

---

## Action items (prevention)

| # | Action | Type (detect/prevent/process) | Owner | Due | Status |
|---|---|---|---|---|---|
| 1 | Add alerting on scheduler job failures | detect | <name> | 2026-06-24 | Open |
| 2 | Add FRA %-used threshold alert at 85% | detect | <name> | 2026-06-24 | Open |
| 3 | Increase FRA size / review retention | prevent | <name> | 2026-06-30 | Open |
| 4 | Add "check job success" to Daily DBA Checklist | process | <name> | 2026-06-20 | Open |

Each action should make recurrence less likely or detection faster. Assign an owner and a
date, and track to closure — a review with no closed actions changes nothing.

---

## Lessons learned

A few sentences capturing the durable takeaway for the team and future-you. Link to any
runbook updated as a result (e.g., "added FRA reclaim steps to Space Management
Procedure.md").

---

## Blameless principle

Describe **systems and processes**, not individuals. People act reasonably given the
information and tools they had; if an action was wrong, ask what made it look right at the
time. The goal is a more resilient system, and a culture where people surface problems
early instead of hiding them.
