# Project 3 — RetentionOps MCP
**Design Preface (no implementation yet)**  
**Owner:** François Tilkin  
**Builds on:** Project 1 (Revenue Monitoring) + Project 2 (Customer Retention Ownership: SQL1→SQL5)

## What this is
Most companies have dashboards and alerts. What they lack is the **execution layer**: turning a reliable signal into a repeatable action workflow with exports, tickets, and audit.

**RetentionOps MCP** is a design for a closed-loop retention workflow:
**Detect → Package → Execute → Log**  
Business logic stays in SQL (auditable). MCP orchestrates actions (standard interface).

---

## Problem (business)
Even with good detection, teams lose time and miss actions because the workflow is manual:
- review dashboards
- decide who to target
- export lists
- write briefs
- post to Slack
- create tickets
- no audit trail

Result: inconsistent response and slow time-to-action during retention incidents.

---

## Solution (high level)
A thin MCP layer that reads existing SQL outputs and generates an **Action Pack**:
- **Manager-ready brief** (Slack/email format)
- **Priority audiences** (P1/P2/P3) exported to CSV or Google Sheets
- **Playbook execution** via Make.com (Slack + ticket + artifact links)
- **Audit log** written to BigQuery

No ML. No “AI magic”. Rule-based, explainable, and safe to disable.

---

## Architecture

'''
SQL1-5 (existing) Execution (no-code)
┌───────────────────────┐ ┌───────────────────────────────┐
│ SQL5: alert_logic │ alert │ Make.com webhook │
│ (AT_RISK% vs baseline) ├──────────► │ - Slack post │
└───────────┬───────────┘ │ - Jira/Asana ticket │
│ │ - Google Sheet upload │
│ context + audiences │ - Store artifact links │
┌───────────▼───────────┐ └───────────────┬───────────────┘
│ MCP Server (new) │ │
│ - reads SQL tables │ │
│ - builds Action Pack │ │
│ - exports P1/P2/P3 │ │
│ - writes logs │ │
└───────────┬───────────┘ │
│ │
┌───────────▼───────────┐ │
│ BigQuery (new tables) │◄───────────────────────────┘
│ - customer_priority │
│ - ops_runs_log │
└───────────────────────┘
'''
---

## Inputs (already available from Project 2)
Read-only:
- `alert_logic` (SQL5): `alert_flag, severity, current_value, baseline_value, delta_relative_pct, sample_size`
- `retention_snapshot` (SQL2): `customer_id, retention_status, days_since_last_order`
- `churn_detection` (SQL3): `customer_id, churn_risk_level, churn_risk_score, signals...`
- `base_customers` (SQL1): `customer_id, total_revenue, total_orders, avg_order_value, last_order_date`

New tables added on top (no refactor):
- `customer_priority` (daily)
- `ops_runs_log` (audit trail)

---

## Priority logic (explainable)
Goal: when time/budget is limited, contact the right customers first.

**P1 — Prevention (highest ROI)**
- `ACTIVE` + `HIGH` risk (+ optionally higher value)

**P2 — Intervention**
- `AT_RISK` + (`HIGH` or `MEDIUM` risk)

**P3 — Win-back**
- `INACTIVE` + `HIGH` risk (optionally recent)

A simple `priority_score (0–100)` ranks customers within each tier using value + urgency signals.
No black box.

---

## MCP tools (v1)
1) `get_alert_decision(date)`  
Reads SQL5 alert decision.

2) `get_operational_context(date)`  
Status distribution + risk distribution.

3) `build_priority_queue(date)`  
Creates/updates `customer_priority` (P1/P2/P3 + scores).

4) `generate_brief(date)`  
Produces Slack/email-ready briefing text.

5) `export_audience(date, tier, destination="csv|gsheet")`  
Exports P1/P2/P3 lists.

6) `trigger_playbook(date, severity)`  
Calls Make.com webhook and writes `ops_runs_log`.

---

## What gets delivered (artifacts)
- `briefing_warning.md` (or email template)
- `audience_P1_YYYY-MM-DD.csv` (+ P2/P3)
- `ops_run_log.json` (example audit record)
- Links to Looker dashboard and exported sheets

---

## Safeguards (built-in)
- Keep alerts macro-level: **one trusted metric** (AT_RISK% deviation) to avoid alert fatigue
- Baseline smoothing: rolling average reduces daily noise
- Anti-noise gates: minimum sample size + baseline warmup
- Safe disable: turning off MCP does not affect SQL pipelines

---

## 90-second demo (planned)
1) Read alert decision for a date (SQL5)
2) Generate manager brief (Slack format)
3) Build priority queue P1/P2/P3
4) Export P1 + P2 to Google Sheets
5) Trigger playbook (Slack + ticket + log)

---

## Limitations (honest)
- UCI dataset lacks contact channels → exports are `customer_id`-based; CRM mapping assumed
- No campaign events → impact tracking is defined as a **conversion proxy** (repurchase within 30 days) for future v2

---

## Why a recruiter should care
This design shows:
- business-first thinking (detection is useless without execution)
- production mindset (safeguards, auditability, rollback)
- interoperability approach (MCP as standard interface)
- scalability and tool-agnostic architecture (BigQuery/Snowflake, Make/Zapier)

---

## Next
Full design spec (long form): `docs/design.md`  
Example outputs (mock artifacts): `artifacts/`

