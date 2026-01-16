# RetentionOps MCP — Design Document (Long Form)

Owner: François Tilkin  
Status: Design complete (not implemented)  
Scope: Project 3 closes the loop of Projects 1 & 2 (SQL-based detection → operational execution)

---

## 0. Intent in one sentence
Turn a trusted retention incident signal into a repeatable, auditable, ready-to-execute action workflow (brief + prioritized audiences + playbook + logs), without changing the existing SQL detection system.

---

## 1. Background
### 1.1 What already exists (Projects 1 & 2)
Project 2 provides an auditable SQL pipeline:

- SQL1 `base_customers` (facts, 1 row/customer)
- SQL2 `retention_snapshot` (time-dependent status: ACTIVE / AT_RISK / INACTIVE)
- SQL3 `churn_detection` (rule-based risk score, explainable signals, risk level)
- SQL4 `cohort_retention_evolution` (cohort retention curves, strategic trend)
- SQL5 `alert_logic` (macro alert on AT_RISK% deviation vs rolling baseline)

Project 1 provides the analytics foundation (BigQuery + Looker Studio + basic automations).

### 1.2 The execution gap (the real problem)
Teams often have detection and dashboards, but execution is manual and inconsistent:
- interpret signal
- decide priority
- export audiences
- write a brief
- notify stakeholders
- create tickets
- no audit trail
- no consistent response cadence

This design addresses that gap.

---

## 2. Goals and non-goals
### 2.1 Goals
1) Convert SQL5 alert output into a consistent “Action Pack”  
2) Provide explainable prioritization (P1/P2/P3) using existing fields  
3) Automate operational delivery (Slack/email + ticket + export links) via Make.com  
4) Ensure traceability (logs + artifacts)  
5) Keep changes low-risk: no modifications to SQL1–SQL5 logic

### 2.2 Non-goals (explicitly out of scope in v1)
- Multi-touch attribution, open/click tracking
- Offer optimization or personalized discount engine
- Real-time streaming and sub-daily execution
- UI productization, auth/roles, multi-tenant platform
- Machine learning churn probability model (this system is rule-based by design)

---

## 3. High-level solution
A thin MCP server layer orchestrates:
- read SQL tables
- create a priority queue table
- generate a manager-ready brief
- export audiences
- trigger a no-code playbook (Make.com)
- write execution logs

Key principle: SQL remains the source of truth; MCP is an interoperability/orchestration layer.

---

## 4. Architecture

```
Detection & Scoring (existing SQL) Orchestration & Execution
┌──────────────────────────────┐            ┌─────────────────────────────────┐
│     SQL5: alert_logic        │   alert    │      MCP Server (new)           │
│      - alert_flag            ├──────────► │       - reads SQL outputs       │
│      - severity              │            │       - builds Action Pack      │
│      - deltas vs baseline    │            │       - exports P1/P2/P3        │
└───────────────┬──────────────┘            │       - triggers Make webhook   │
                │                           │       - writes audit logs       │
                │  context + priority       └────────────────┬────────────────┘
┌───────────────▼──────────────┐                             │
│   SQL2: retention_snapshot   │                             │ 
│   SQL3: churn_detection      │                             │
│   SQL1: base_customers       │                             │
└───────────────┬──────────────┘                             │
                │                                            │
                │                                            │
┌───────────────▼───────────────┐                            │
│     BigQuery (new tables)     │◄───────────────────────────┘
│      - customer_priority      │
│      - ops_runs_log           │
└───────────────────────────────┘
```
Make.com (no-code execution)

Slack post

Jira/Asana ticket

Google Sheets upload

store artifact links


---

## 5. Data inputs (read-only)
### 5.1 `alert_logic` (SQL5)
Required fields (already present):
- `alert_date`
- `severity` (INFO/WARNING/CRITICAL)
- `alert_flag` (TRUE/FALSE)
- `current_value`
- `baseline_value`
- `delta_relative_pct`
- `sample_size`

### 5.2 `retention_snapshot` (SQL2)
Required fields:
- `customer_id`
- `retention_status` (ACTIVE/AT_RISK/INACTIVE)
- `days_since_last_order`

Note: in current implementation, SQL2 materializes “today” only. v1 assumes “as-of today” for priority runs.

### 5.3 `churn_detection` (SQL3)
Required fields:
- `customer_id`
- `churn_risk_score`
- `churn_risk_level` (LOW/MEDIUM/HIGH)
- `is_frequency_drop`
- `is_status_inconsistent`
- `is_value_drop` (disabled in v1 scoring but field exists)

### 5.4 `base_customers` (SQL1)
Required fields:
- `customer_id`
- `total_revenue`
- `total_orders`
- `avg_order_value`
- `last_order_date`

---

## 6. New tables (added on top; no refactor)
### 6.1 `customer_priority`
Purpose: daily prioritized queue derived from existing tables.

Schema (proposed):
```sql
CREATE TABLE customer_priority (
  run_date DATE NOT NULL,
  customer_id STRING NOT NULL,

  retention_status STRING NOT NULL,
  churn_risk_level STRING NOT NULL,
  churn_risk_score INT64,

  priority_tier STRING NOT NULL,        -- P1/P2/P3
  priority_score FLOAT64 NOT NULL,      -- 0–100
  priority_reason STRING NOT NULL,      -- short explanation

  days_since_last_order INT64,
  total_revenue NUMERIC,
  total_orders INT64,
  avg_order_value NUMERIC,

  recommended_action STRING,            -- short, rule-based
  value_proxy NUMERIC,                  -- simple estimate

  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP()
)
PARTITION BY run_date
CLUSTER BY priority_tier, priority_score DESC;

Retention: 90 days in portfolio context (configurable).

6.2 ops_runs_log

Purpose: audit trail of every orchestration run.

Schema (proposed):

CREATE TABLE ops_runs_log (
  run_id STRING NOT NULL,
  run_timestamp TIMESTAMP NOT NULL,

  alert_date DATE,
  severity STRING,
  alert_flag BOOLEAN,

  current_value FLOAT64,
  baseline_value FLOAT64,
  delta_relative_pct FLOAT64,
  sample_size INT64,

  p1_count INT64,
  p2_count INT64,
  p3_count INT64,
  total_exported INT64,

  slack_posted BOOLEAN,
  slack_message_ts STRING,
  ticket_id STRING,

  export_p1_link STRING,
  export_p2_link STRING,
  export_p3_link STRING,

  status STRING,                 -- success/failed/skipped
  error_message STRING,
  execution_time_ms INT64,

  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP()
)
PARTITION BY DATE(run_timestamp)
CLUSTER BY alert_date, severity;

7. Priority logic (P1/P2/P3)
7.1 Priority tiers (v1)

Goal: simple, explainable, aligned with operational ROI.

P1 (Prevention): ACTIVE + HIGH

P2 (Intervention): AT_RISK + (HIGH or MEDIUM)

P3 (Win-back): INACTIVE + HIGH (optionally recent if business wants)

Optional value filter:

P1 can be constrained by value proxy (e.g., above median total_revenue) to keep volume manageable.

7.2 Priority score (0–100)

This is a ranking helper, not a black box.

Example scoring:

Base by tier: P1=70, P2=50, P3=30

Value bonus (0–20): min(20, ln(1 + total_revenue) * 2)

Urgency bonus (0–10): min(10, days_since_last_order / 3)

Total = base + value + urgency (cap at 100).

7.3 Recommended action (rule-based)

Simple mapping; no personalization engine in v1.

Examples:

P1: “VIP outreach + light incentive”

P2: “win-back sequence + shipping / discount”

P3: “strong comeback offer or survey”

8. MCP server design
8.1 Why MCP here

MCP is used as a clean interface layer so orchestration logic is tool-agnostic (can swap BigQuery/Slack/Jira providers without rewriting business logic). It also demonstrates production-style separation: SQL logic is stable; orchestration is a layer.

8.2 Tools (v1)

get_alert_decision(date)

Reads SQL5 row for date

Returns the decision payload

get_operational_context(date)

Produces:

status distribution from retention_snapshot

risk distribution by tier from churn_detection joined with status

counts for P1/P2/P3 (preview)

build_priority_queue(date)

Writes customer_priority for date using joins:

retention_snapshot + churn_detection + base_customers

Returns counts

generate_brief(date, format="slack|email")

Produces a short, manager-ready message using:

alert decision (SQL5)

operational context

export_audience(date, tier, destination="csv|gsheet", limit=2000)

Exports rows from customer_priority filtered by tier

Returns artifact link(s)

trigger_playbook(date)

Calls Make webhook with:

brief text

alert decision

tier counts

artifact links

Writes ops_runs_log

8.3 Tools (future v2)

get_recommended_action(customer_id) (requires action_catalog)

measure_impact(execution_id, horizon_days=30) (requires campaign events or proxy)

9. Make.com playbook (execution layer)
9.1 Trigger

Webhook called by MCP with payload:

date, severity, alert_flag

brief text

tier counts

export links (if already produced) or instruction to fetch exports

9.2 WARNING playbook

Post to Slack channel #retention-alerts

Create ticket (priority HIGH)

Upload P1 and P2 to Google Sheets

Update log / return artifact links

9.3 CRITICAL playbook

Post to Slack with escalation mention

Create urgent ticket

Upload P1/P2/P3

Update log

10. Deliverables (artifacts)
10.1 In-repo example artifacts (static mocks)

Place in /artifacts/:

example_brief_warning.md

example_audience_p1.csv

example_run_log.json

These allow a recruiter to understand the outputs without running code.

10.2 Production artifacts (when implemented)

Slack message

Ticket link

Sheet/CSV exports

BigQuery log row

11. Demo plan (90 seconds)

Goal: show closed-loop thinking, not code complexity.

Read alert decision (SQL5)

Generate brief

Build priority queue table

Export P1 and P2

Trigger playbook (Slack + ticket + log)

12. Assumptions and limitations (honest)
12.1 Dataset limitation (portfolio context)

UCI Online Retail lacks:

contact channels (email/phone)

campaign events (open/click)

campaign costs

Mitigation:

exports are customer_id based; real companies map to CRM contacts

impact measurement in v1 is specified as a proxy: repurchase within 30 days

12.2 Privacy / tracking statement (avoid brittle claims)

Privacy changes reduce tracking precision and increase the value of first-party data. This system focuses on first-party behavioral signals from transaction history and does not require cross-site tracking.

13. Impact model (illustrative, not guaranteed)

This is a design document. Any € impact is scenario-based.

Define variables:

N = number of customers in targeted tiers

C = conversion rate uplift from intervention (assumption)

V = value proxy per customer (median revenue or AOV-based proxy)

Illustrative revenue protected = N * C * V

Provide sensitivity table (example template):

C = 10% / 20% / 30%

V = median total_revenue or selected proxy

Document assumptions clearly.

14. Error handling (v1)

Principles:

fail safe (no partial silent failures)

logs always written (even on error)

retry only for transient failures

Examples:

BigQuery timeout: retry 3 times with backoff

webhook failure: retry 2 times; fallback to writing log + outputting brief + links

export failure: keep CSV stored and link to it in brief

15. Monitoring (meta-monitoring)

Measure system health:

daily run success rate

webhook failure rate

export success rate

average execution time

alert frequency (target: low single digits per month; validate during pilot)

16. Rollout plan (if implemented)

Shadow mode: generate outputs, do not post to main channels

Pilot: WARNING only, limited channel

Production: WARNING + CRITICAL with escalation

Each phase has a rollback:

disable MCP server; SQL pipelines continue unchanged

17. Open questions (explicit)

These are not blockers for the design doc, but should be clarified for real implementation:

Where to store exports (GCS vs Sheets only vs both)?

Ticketing system of choice (Jira/Asana/Trello)?

Does business want a value threshold for P1?

Do we require “recent INACTIVE” for P3?

SLA expectations for response time and escalation ownership?

18. Appendix
18.1 Data lineage (conceptual)

orders → base_customers → retention_snapshot → churn_detection
retention_snapshot → alert_logic
base_customers + retention_snapshot + churn_detection → customer_priority
alert_logic + customer_priority → playbook outputs + ops_runs_log

18.2 Implementation boundaries

This document intentionally separates:

detection/scoring (SQL; stable)

orchestration (MCP; replaceable)

execution (Make; replaceable)

measurement (future v2)

End of design doc.
