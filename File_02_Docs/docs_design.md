# RetentionOps MCP — Design Document

**Owner:** François Tilkin  
**Status:** Design Complete (Not Implemented)  
**Type:** Architecture Proposal  
**Scope:** Project 3 closes the execution gap between Projects 1 & 2 (SQL-based detection → operational action)

---

## Executive Summary

**Intent in one sentence:**  
Turn a trusted retention incident signal into a repeatable, auditable, ready-to-execute action workflow (brief + prioritized audiences + playbook + logs), without changing the existing SQL detection system.

**Business problem:**  
Teams have detection (SQL5 alerts) and dashboards but execution is manual and inconsistent: interpret signal, decide priority, export audiences, write brief, notify stakeholders, create tickets. No audit trail, no consistent cadence. Result: 60% of alerts ignored due to time pressure.

**Solution:**  
Thin MCP orchestration layer that reads SQL outputs, builds priority queues (P1/P2/P3), generates manager-ready briefs, exports CRM-ready audiences, triggers Make.com workflows, and writes complete audit logs.

**Key principle:**  
SQL detection logic remains unchanged and trusted. MCP adds operational execution layer only.

---

## Table of Contents

### Context
1. [Background](#1-background)
2. [Goals and Non-Goals](#2-goals-and-non-goals)
3. [High-Level Solution](#3-high-level-solution)

### Architecture
4. [System Architecture](#4-system-architecture)
5. [Data Inputs (Read-Only)](#5-data-inputs-read-only)
6. [New Tables (Added on Top)](#6-new-tables-added-on-top)

### Business Logic
7. [Priority Logic (P1/P2/P3)](#7-priority-logic-p1p2p3)
8. [MCP Server Design](#8-mcp-server-design)
9. [Make.com Playbook](#9-makecom-playbook)

### Deliverables
10. [Artifacts](#10-artifacts)
11. [Demo Plan (90 Seconds)](#11-demo-plan-90-seconds)

### Risk Management
12. [Assumptions and Limitations](#12-assumptions-and-limitations)
13. [Impact Model](#13-impact-model)
14. [Error Handling](#14-error-handling)
15. [Monitoring](#15-monitoring)

### Implementation
16. [Rollout Plan](#16-rollout-plan)
17. [Open Questions](#17-open-questions)
18. [Appendix](#18-appendix)

---

## 1. Background

### 1.1 What Already Exists (Projects 1 & 2)

**Project 2** provides an auditable SQL pipeline:

| Module | Table | Purpose |
|--------|-------|---------|
| SQL1 | `base_customers` | Customer facts (1 row/customer) |
| SQL2 | `retention_snapshot` | Time-dependent status (ACTIVE/AT_RISK/INACTIVE) |
| SQL3 | `churn_detection` | Rule-based risk scoring + explainable signals |
| SQL4 | `cohort_retention_evolution` | Cohort retention curves (strategic trends) |
| SQL5 | `alert_logic` | Macro alert on AT_RISK% deviation vs baseline |

**Project 1** provides analytics foundation:
- BigQuery data warehouse
- Looker Studio dashboards
- Basic Make.com automations (daily revenue alerts)

---

### 1.2 The Execution Gap (The Real Problem)

**Current manual process:**

| Step | Tool | Who | Time | Issue |
|------|------|-----|------|-------|
| 1. Detect incident | SQL5 alert | System | Auto | ✅ Works |
| 2. Review dashboard | Looker | Analyst | 45 min | Manual |
| 3. Identify high-risk | SQL3 query | Analyst | 20 min | Manual |
| 4. Decide priorities | Spreadsheet | Manager | 30 min | Manual |
| 5. Write brief | Email | Analyst | 15 min | Manual |
| 6. Export lists | BigQuery | Analyst | 20 min | Manual |
| 7. Notify team | Slack | Analyst | 10 min | Manual |
| 8. Create ticket | Jira | Manager | 15 min | Manual |
| 9. Track actions | Nothing | Nobody | 0 min | ❌ Lost |
| 10. Measure impact | Nothing | Nobody | 0 min | ❌ Never |

**Total overhead:** 2h 55min per alert

**Actual response rate:** 40% (alerts ignored due to time pressure)

**This design addresses that gap.**

---

## 2. Goals and Non-Goals

### 2.1 Goals

1. **Convert SQL5 alert output into consistent "Action Pack"**  
   - Manager-ready brief
   - Prioritized audience exports (P1/P2/P3)
   - Automated notifications
   - Complete audit trail

2. **Provide explainable prioritization** using existing fields  
   - No ML black boxes
   - Rule-based, auditable logic
   - Every decision explainable in 5 seconds

3. **Automate operational delivery** via Make.com  
   - Slack/email notifications
   - Jira ticket creation
   - Google Sheets exports

4. **Ensure traceability**  
   - Every run logged
   - Every artifact linked
   - Regulatory compliance ready

5. **Keep changes low-risk**  
   - No modifications to SQL1-5 logic
   - MCP reads only, never modifies detection
   - Can disable without breaking existing system

---

### 2.2 Non-Goals (Explicitly Out of Scope in V1)

❌ Multi-touch attribution, open/click tracking  
❌ Offer optimization or personalized discount engine  
❌ Real-time streaming (daily batch sufficient)  
❌ UI productization, auth/roles, multi-tenant  
❌ Machine learning churn probability model (intentionally rule-based)

**Why out of scope:**  
These require data not available in UCI Online Retail dataset (portfolio context). Real company implementation can add these as V2 features.

---

## 3. High-Level Solution

### Design Principle: Thin Orchestration Layer

**MCP server orchestrates:**
- Read SQL tables (detection outputs)
- Create priority queue table
- Generate manager-ready brief
- Export audiences (CSV/Google Sheets)
- Trigger no-code playbook (Make.com)
- Write execution logs

**Key principle:**  
SQL remains source of truth. MCP is interoperability/orchestration layer only.

**Tool-agnostic design:**
- BigQuery → Swap to Snowflake (change connector)
- Make.com → Swap to Zapier (change webhook)
- Slack → Swap to Teams (change message format)

---

## 4. System Architecture

```
┌──────────────────────────────────────────────────────────────┐
│          DETECTION & SCORING (Existing SQL — Projects 1 & 2) │
│                                                              │
│  SQL5: alert_logic                                           │
│  ├─ alert_flag (TRUE/FALSE)                                  │
│  ├─ severity (WARNING/CRITICAL)                              │
│  └─ deltas vs baseline                                       │
│                                                              │
│  Context Tables:                                             │
│  ├─ SQL2: retention_snapshot (status)                        │
│  ├─ SQL3: churn_detection (risk)                             │
│  └─ SQL1: base_customers (facts)                             │
└────────────────────────┬─────────────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────────────────┐
│          ORCHESTRATION (MCP Server — NEW)                    │
│                                                              │
│  MCP Tools:                                                  │
│  1. get_alert_decision(date)                                 │
│  2. get_operational_context(date)                            │
│  3. build_priority_queue(date)                               │
│  4. generate_brief(date, format)                             │
│  5. export_audience(date, tier, destination)                 │
│  6. trigger_playbook(date)                                   │
│                                                              │
│  New Tables:                                                 │
│  ├─ customer_priority (P1/P2/P3 assignments)                 │
│  └─ ops_runs_log (audit trail)                               │
└────────────────────────┬─────────────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────────────────┐
│          EXECUTION (Make.com — No-Code Workflows)            │
│                                                              │
│  WARNING Playbook:                                           │
│  ├─ Post brief to Slack (#retention-alerts)                  │
│  ├─ Create Jira ticket (Priority: High)                      │
│  ├─ Upload P1 + P2 to Google Sheets                          │
│  └─ Write execution log                                      │
│                                                              │
│  CRITICAL Playbook:                                          │
│  ├─ Post brief + @channel escalation                         │
│  ├─ Create urgent Jira ticket (Priority: Highest)            │
│  ├─ Upload P1 + P2 + P3 to Google Sheets                     │
│  └─ Write execution log                                      │
└──────────────────────────────────────────────────────────────┘
```

---

## 5. Data Inputs (Read-Only)

### 5.1 `alert_logic` (SQL5)

**Purpose:** Macro alert decision on AT_RISK% deviation

**Required fields:**
- `alert_date` (DATE)
- `severity` (STRING: INFO/WARNING/CRITICAL)
- `alert_flag` (BOOLEAN)
- `current_value` (FLOAT64: current AT_RISK%)
- `baseline_value` (FLOAT64: 7-day average)
- `delta_relative_pct` (FLOAT64: % increase)
- `sample_size` (INT64)

**Query pattern:**
```sql
SELECT * FROM alert_logic WHERE alert_date = @date
```

---

### 5.2 `retention_snapshot` (SQL2)

**Purpose:** Time-dependent customer status classification

**Required fields:**
- `customer_id` (STRING)
- `retention_status` (STRING: ACTIVE/AT_RISK/INACTIVE)
- `days_since_last_order` (INT64)

**Note:** Current implementation materializes "today" only. V1 assumes "as-of today" for priority runs.

**Query pattern:**
```sql
SELECT customer_id, retention_status, days_since_last_order
FROM retention_snapshot
```

---

### 5.3 `churn_detection` (SQL3)

**Purpose:** Behavioral risk scoring with explainable signals

**Required fields:**
- `customer_id` (STRING)
- `churn_risk_score` (INT64: 0-70 in V1)
- `churn_risk_level` (STRING: LOW/MEDIUM/HIGH)
- `is_frequency_drop` (BOOLEAN)
- `is_status_inconsistent` (BOOLEAN)
- `is_value_drop` (BOOLEAN: disabled in V1)

**Query pattern:**
```sql
SELECT customer_id, churn_risk_level, churn_risk_score,
       is_frequency_drop, is_status_inconsistent
FROM churn_detection
```

---

### 5.4 `base_customers` (SQL1)

**Purpose:** Customer facts (1 row per customer)

**Required fields:**
- `customer_id` (STRING)
- `total_revenue` (NUMERIC)
- `total_orders` (INT64)
- `avg_order_value` (NUMERIC)
- `last_order_date` (DATE)

**Query pattern:**
```sql
SELECT customer_id, total_revenue, total_orders, avg_order_value
FROM base_customers
```

---

## 6. New Tables (Added on Top; No Refactor)

### 6.1 `customer_priority`

**Purpose:** Daily prioritized queue derived from existing tables

**Schema:**
```sql
CREATE TABLE customer_priority (
  run_date DATE NOT NULL,
  customer_id STRING NOT NULL,

  -- Context from input tables
  retention_status STRING NOT NULL,
  churn_risk_level STRING NOT NULL,
  churn_risk_score INT64,

  -- Priority assignment
  priority_tier STRING NOT NULL,        -- P1/P2/P3
  priority_score FLOAT64 NOT NULL,      -- 0–100 ranking
  priority_reason STRING NOT NULL,      -- Short explanation

  -- Customer context
  days_since_last_order INT64,
  total_revenue NUMERIC,
  total_orders INT64,
  avg_order_value NUMERIC,

  -- Action recommendation
  recommended_action STRING,            -- Rule-based action
  value_proxy NUMERIC,                  -- Estimated value at risk

  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP()
)
PARTITION BY run_date
CLUSTER BY priority_tier, priority_score DESC;
```

**Refresh:** Daily (after SQL1-5 refresh)

**Retention:** 90 days (portfolio context; configurable for production)

---

### 6.2 `ops_runs_log`

**Purpose:** Audit trail of every orchestration run

**Schema:**
```sql
CREATE TABLE ops_runs_log (
  -- Run identification
  run_id STRING NOT NULL,
  run_timestamp TIMESTAMP NOT NULL,
  alert_date DATE,

  -- Alert context
  severity STRING,
  alert_flag BOOLEAN,
  current_value FLOAT64,
  baseline_value FLOAT64,
  delta_relative_pct FLOAT64,
  sample_size INT64,

  -- Priority queue results
  p1_count INT64,
  p2_count INT64,
  p3_count INT64,
  total_exported INT64,

  -- Execution artifacts
  slack_posted BOOLEAN,
  slack_message_ts STRING,
  ticket_id STRING,
  export_p1_link STRING,
  export_p2_link STRING,
  export_p3_link STRING,

  -- Execution status
  status STRING,                        -- success/failed/skipped
  error_message STRING,
  execution_time_ms INT64,

  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP()
)
PARTITION BY DATE(run_timestamp)
CLUSTER BY alert_date, severity;
```

**Refresh:** Written on every run

**Retention:** Permanent (regulatory compliance)

---

## 7. Priority Logic (P1/P2/P3)

### 7.1 Priority Tiers (V1)

**Goal:** Simple, explainable, aligned with operational ROI

#### **P1 (Prevention) — Highest ROI**

**Criteria:**
```
retention_status = 'ACTIVE'
AND churn_risk_level = 'HIGH'
AND (optional: total_revenue >= median)
```

**Why first:**
- Still buying (easier to save)
- High risk (will churn if ignored)
**Expected conversion: To be measured during pilot**

**Action:** VIP outreach + light incentive

**Expected volume:** 80-100 customers

---

#### **P2 (Intervention) — Rescue Mission**

**Criteria:**
```
retention_status = 'AT_RISK'
AND churn_risk_level IN ('HIGH', 'MEDIUM')
```

**Why second:**
- Already sliding (31-90 days dormant)
- Still winnable
**Expected conversion: To be measured during pilot**

**Action:** Win-back sequence + shipping/discount

**Expected volume:** 300-400 customers

---

#### **P3 (Win-back) — Last Chance**

**Criteria:**
```
retention_status = 'INACTIVE'
AND churn_risk_level = 'HIGH'
AND (optional: days_since_last_order <= 90)
```

**Why third:**
- Already churned (90+ days)
- Recent enough to remember brand
**Expected conversion: To be measured during pilot**

**Action:** Strong comeback offer or survey

**Expected volume:** 150-200 customers

---

### 7.2 Priority Score (0–100)

**Purpose:** Rank customers within each tier

**Formula:**
```
priority_score = base + value_bonus + urgency_bonus
```

**Components:**

**Base by tier:**
- P1 = 70 points
- P2 = 50 points
- P3 = 30 points

**Value bonus (0–20 points):**
```python
value_bonus = min(20, ln(1 + total_revenue) * 2)
```

**Urgency bonus (0–10 points):**
```python
urgency_bonus = min(10, days_since_last_order / 3)
```

**Total:** Cap at 100 points

**Example:**
- Customer: ACTIVE + HIGH risk + €3,847 revenue + 24 days dormant
- Score: 70 + 16.9 + 8.0 = **94.9 points** → Top of P1 queue

---

### 7.3 Recommended Action (Rule-Based with Value Segmentation)

**P1 Actions vary by customer value (explainable sub-segmentation):**

| Value Segment | Threshold | Action | Rationale |
|---------------|-----------|--------|-----------|
| High-value | >€2,000 | VIP outreach + 10% loyalty incentive | Highest ROI, personalized touch |
| Medium-value | €1,000-2,000 | Personal email + free shipping offer | Balance personalization + cost |
| Standard-value | <€1,000 | Personal email + reminder | Cost-effective prevention |

**P2 Actions:**
- Win-back email sequence (3 emails, 7 days) + free shipping

**P3 Actions:**
- Strong comeback offer (20% discount) + survey

**Implementation note:** V1 uses value-based segmentation within P1 to optimize ROI while remaining rule-based and explainable. This avoids the "one-size-fits-all" approach while staying auditable (no ML black box).

---

## 8. MCP Server Design

### 8.1 Why MCP Here

**MCP provides:**
- Clean interface layer (tool-agnostic)
- Production-style separation (SQL stable, orchestration replaceable)
- Standard protocol (industry-emerging standard)

**Benefits:**
- Swap BigQuery → Snowflake (just change connector)
- Swap Make.com → Zapier (just change webhook)
- Demonstrates architectural maturity

---

### 8.2 Tools (V1)

#### **Tool 1: `get_alert_decision(date)`**

**Purpose:** Read SQL5 alert decision

**Input:**
```json
{"date": "2026-01-12"}
```

**Output:**
```json
{
  "alert_date": "2026-01-12",
  "severity": "WARNING",
  "alert_flag": true,
  "current_value": 25.2,
  "baseline_value": 21.4,
  "delta_relative_pct": 18.0,
  "sample_size": 49378
}
```

---

#### **Tool 2: `get_operational_context(date)`**

**Purpose:** Aggregate status and risk distributions

**Input:**
```json
{"date": "2026-01-12"}
```

**Output:**
```json
{
  "total_customers": 49378,
  "status_distribution": {
    "ACTIVE": 19012,
    "AT_RISK": 12450,
    "INACTIVE": 17916
  },
  "at_risk_by_risk_level": {
    "HIGH": 2891,
    "MEDIUM": 5178,
    "LOW": 4381
  }
}
```

---

#### **Tool 3: `build_priority_queue(date)`**

**Purpose:** Calculate P1/P2/P3 assignments and write to `customer_priority`

**Input:**
```json
{"date": "2026-01-12"}
```

**Output:**
```json
{
  "p1_count": 87,
  "p2_count": 312,
  "p3_count": 189,
  "total": 588,
  "table_updated": "customer_priority"
}
```

**Logic:** Joins `retention_snapshot` + `churn_detection` + `base_customers`, applies priority tier rules, calculates scores.

---

#### **Tool 4: `generate_brief(date, format)`**

**Purpose:** Create manager-ready briefing

**Input:**
```json
{
  "date": "2026-01-12",
  "format": "slack"
}
```

**Output:** Markdown text (Slack/email ready)

**Content:**
- Alert severity and metrics
- Status distribution
- Risk breakdown
- Actions ready (P1/P2/P3 counts)
- Links (dashboard, exports, ticket)

---

#### **Tool 5: `export_audience(date, tier, destination, limit)`**

**Purpose:** Export priority tier to CSV or Google Sheets

**Input:**
```json
{
  "date": "2026-01-12",
  "tier": "P1",
  "destination": "gsheet",
  "limit": 2000
}
```

**Output:**
```json
{
  "rows_exported": 87,
  "gsheet_url": "https://docs.google.com/spreadsheets/d/..."
}
```

**Destinations:** `csv` | `gsheet` | `both`

---

#### **Tool 6: `trigger_playbook(date)`**

**Purpose:** Execute Make.com workflow and write audit log

**Input:**
```json
{"date": "2026-01-12"}
```

**Output:**
```json
{
  "webhook_called": true,
  "response_status": 200,
  "artifacts": {
    "slack_posted": true,
    "jira_ticket": "RET-2401",
    "gsheets_uploaded": true
  },
  "log_written": true
}
```

**Workflow:**
1. Gather all data (alert, context, brief, exports)
2. Call Make.com webhook
3. Wait for response (Slack timestamp, Jira ticket ID)
4. Write audit log to `ops_runs_log`

---

### 8.3 Tools (Future V2)

**Tool 7: `get_recommended_action(customer_id)`**  
Requires: `action_catalog` table

**Tool 8: `measure_impact(execution_id, horizon_days=30)`**  
Requires: Campaign events or conversion proxy (repurchase tracking)

---

## 9. Make.com Playbook

### 9.1 Webhook Trigger

**URL:** `https://hook.eu1.make.com/xxxxx` (generated by Make.com)

**Method:** POST

**Payload:**
```json
{
  "date": "2026-01-12",
  "severity": "WARNING",
  "alert_flag": true,
  "brief_markdown": "[full text]",
  "metrics": {
    "current_value": 25.2,
    "baseline_value": 21.4,
    "delta_relative_pct": 18.0
  },
  "priority_counts": {
    "p1": 87,
    "p2": 312,
    "p3": 189
  },
  "export_urls": {
    "p1": "https://docs.google.com/...",
    "p2": "https://docs.google.com/..."
  }
}
```

---

### 9.2 WARNING Playbook

**Steps:**

1. **Post to Slack**
   - Channel: `#retention-alerts`
   - Message: `brief_markdown`
   - Store: `slack_message_ts`

2. **Create Jira Ticket**
   - Project: RETENTION
   - Type: Task
   - Priority: High
   - Title: "Retention Alert WARNING - {date}"
   - Store: `ticket_id`

3. **Upload to Google Sheets**
   - P1 audience
   - P2 audience

4. **Write Log**
   - Insert into `ops_runs_log`

**Execution time:** ~10 seconds

---

### 9.3 CRITICAL Playbook

**Differences from WARNING:**

- Slack: Add `@channel` mention
- Jira: Priority **Highest** (not High)
- Exports: Include P3 (not just P1/P2)
- Optional: Auto-trigger email campaign

---

## 10. Artifacts

### 10.1 In-Repo Example Artifacts (For Portfolio)

**Location:** `/artifacts/`

**Files:**
- `example_brief_warning.md` — Manager-ready briefing
- `example_audience_p1.csv` — P1 export sample (10 rows)
- `example_run_log.json` — Execution log sample

**Purpose:** Allow recruiters to understand outputs without running code

---

### 10.2 Production Artifacts (When Implemented)

**Generated on every run:**

1. **Slack message** (posted to channel)
2. **Jira ticket** (with link)
3. **Google Sheets** (P1/P2/P3 audiences)
4. **BigQuery log** (in `ops_runs_log`)

---

## 11. Demo Plan (90 Seconds)

**Goal:** Show closed-loop thinking, not code complexity

**Script:**

1. **[T+0s]** Run: `get_alert_decision(today)`  
   Output: "WARNING, +18% AT_RISK"

2. **[T+5s]** Run: `get_operational_context(today)`  
   Output: Status/risk distributions

3. **[T+12s]** Run: `build_priority_queue(today)`  
   Output: 588 customers → P1/P2/P3

4. **[T+18s]** Run: `generate_brief(today)`  
   Output: Slack-ready markdown

5. **[T+25s]** Run: `export_audience(today, "P1")`  
   Output: Google Sheet with 87 customers

6. **[T+32s]** Run: `trigger_playbook(today)`  
   Output: Slack posted ✅, Jira created ✅, Log written ✅

**[T+35s] COMPLETE**

**What manager sees:** Alert in Slack, P1 list in Sheets, Jira ticket assigned — ready to action in 10 minutes (vs 3 hours manual).

---

## 12. Assumptions and Limitations

### 12.1 Dataset Limitations (Portfolio Context)

**UCI Online Retail dataset lacks:**

| Missing | Impact | Mitigation |
|---------|--------|------------|
| Email/phone | Can't send campaigns directly | Export `customer_id`, CRM maps to contacts |
| Campaign events | Can't track opens/clicks | Use conversion proxy: repurchase within 30 days |
| Marketing costs | Can't calculate precise ROI | Use simplified proxy |
| Multi-market data | UK only, 2010-2011 | Architecture is market-agnostic |

---

### 12.2 Privacy / Tracking Statement

**Approach:** First-party behavioral signals from transaction history only.

**Does not require:** Cross-site tracking, third-party cookies.

**Compliance:** GDPR-friendly (explainable decisions, customer data only).

---

## 13. Impact Model

### Illustrative Revenue Protected Calculation

**This is a design document. Impact is scenario-based.**

**Variables:**
- `N` = Number of customers in targeted tiers
- `C` = Conversion rate uplift from intervention (assumption)
- `V` = Value proxy per customer (median revenue)

**Formula:**
```
Revenue Protected = N × C × V
```

**Scenario Analysis (Sensitivity Table):**

**Note:** Conversion rates are hypothetical assumptions for sizing. Actual rates to be measured during pilot.

| Scenario | P1 Conversion | P2 Conversion | P3 Conversion | Total Revenue Protected |
|----------|---------------|---------------|---------------|-------------------------|
| **Conservative** | 15% | 10% | 5% | €21,234 |
| **Moderate** | 25% | 15% | 8% | €40,781 |
| **Optimistic** | 35% | 20% | 10% | €60,344 |

**Assumptions:**
- Customer counts: P1=87, P2=312, P3=189
- Average customer value: €625 (median from base_customers)
- **Conversion rates are sizing estimates only** (not guaranteed outcomes)

**Measurement approach:** Track repurchase within 30 days post-intervention using `intervention_tracking` table.

---

## 14. Error Handling

### Principles

1. **Fail safe** (no partial silent failures)
2. **Logs always written** (even on error)
3. **Retry only transient failures** (not logic errors)

---

### Error Scenarios

#### **BigQuery Timeout**

**Response:**
- Retry 3 times with exponential backoff (2s, 4s, 8s)
- If all fail: Log error, email data team
- Impact: None (next daily run catches up)

---

#### **Make.com Webhook Failure**

**Response:**
- Retry 2 times (5s apart)
- If fails: Write log, send email backup with brief + export links
- Impact: No Slack/Jira, but email ensures visibility

---

#### **Google Sheets Upload Failure**

**Response:**
- Keep CSV in Cloud Storage
- Include GCS download link in brief
- Impact: Manual download (minor inconvenience)

---

#### **No Baseline Available (Day 1-7)**

**Response:**
- `alert_flag = FALSE`
- `severity = INFO`
- System waits until Day 8
- Impact: None (expected warmup period)

---

## 15. Monitoring

### System Health Metrics

**Track:**
- Daily run success rate (target: >95%)
- Webhook failure rate (target: <5%)
- Export success rate (target: >95%)
- Average execution time (target: <60 seconds)
- Alert frequency (target: 2-3 per month)

**Alerts on:**
- Success rate <90% for 2 consecutive days
- Execution time >120 seconds
- Alert frequency >5/month (too sensitive) or 0 for 60 days (too conservative)

---

## 16. Rollout Plan

### Phase 1: Shadow Mode (Week 4)

**Behavior:**
- System runs daily
- Generates all outputs
- **Nothing posted publicly** (no Slack, no Jira)
- Outputs saved to test folder

**Success criteria:**
- 7 consecutive days with no errors
- CRM manager approves brief format
- Export format works in CRM tool

---

### Phase 2: Pilot (Weeks 5-6)

**Behavior:**
- WARNING alerts live (CRITICAL disabled)
- Posted to test channel `#retention-test`
- Jira tickets in test project
- CRM team launches actual campaigns

**Success criteria:**
- 2 WARNING alerts handled successfully
- Campaign launched within 30 minutes
- No false positives

---

### Phase 3: Full Production (Week 7+)

**Behavior:**
- WARNING and CRITICAL both live
- Posted to main channel `#retention-alerts`
- Jira tickets in production project
- Full team access

---

### Rollback Plan

**Trigger rollback if:**
- False positive rate >5%
- System downtime >24 hours
- Team requests disable

**Procedure:**
```bash
# 1. Disable MCP server
$ systemctl stop mcp-retention-ops

# 2. Pause Make.com workflows (UI)

# 3. Notify team (Slack)

# 4. Document issue, fix, redeploy when ready
```

**Risk:** LOW (MCP only reads, SQL continues unchanged)

---

## 17. Open Questions

### Technical Decisions

**Q1: Export storage strategy?**
- Options: GCS only, Sheets only, or both
- Recommendation: Both (Sheets primary, GCS backup)

**Q2: Ticketing system?**
- Options: Jira, Asana, Trello, Linear
- Recommendation: Jira (most common)

**Q3: P1 value threshold?**
- Options: No threshold, or filter by median revenue
- Recommendation: Start without, add if P1 >150

**Q4: P3 recency filter?**
- Options: 60 days, 90 days, 120 days, or no filter
- Recommendation: 90 days (industry standard)

---

### Business Decisions

**Q5: Alert escalation SLA?**
- WARNING response time: 24 hours?
- CRITICAL response time: 4 hours?
- Who owns escalation?

**Q6: Campaign response expectations?**
- Who executes campaigns? (CRM? Marketing?)
- What if export lists too large?
- Acceptable response time?

**Q7: Success measurement?**
- What KPIs define success?
- How long should pilot run?
- Rollback threshold?

---

## 18. Appendix

### 18.1 Data Lineage (Conceptual)

```
orders
  ↓
base_customers (SQL1)
  ↓
  ├─→ retention_snapshot (SQL2)
  │     ↓
  │     ├─→ churn_detection (SQL3)
  │     │     ↓
  │     │     └─→ customer_priority (NEW)
  │     │
  │     └─→ alert_logic (SQL5)
  │           ↓
  │           └─→ MCP Server
  │                 ↓
  │                 └─→ ops_runs_log (NEW)
  │
  └─→ cohort_retention_evolution (SQL4)
```

---

### 18.2 Implementation Boundaries

**This document intentionally separates:**

| Layer | Technology | Changeability |
|-------|------------|---------------|
| Detection/Scoring | SQL (SQL1-5) | Stable, trusted |
| Orchestration | MCP | Replaceable |
| Execution | Make.com | Replaceable |
| Measurement | Future V2 | To be designed |

**Design philosophy:** Each layer independent, swappable without affecting others.

---

## End of Design Document

**Next steps if approved:**
1. Stakeholder review (CRM, Marketing, Data teams)
2. Technical validation (Data Engineering review)
3. Budget approval
4. 3-week implementation sprint
5. Shadow mode → Pilot → Production rollout

**Questions:** Contact François Tilkin

---

*Last updated: 2026-01-17*  
*Version: 1.0*  
*Status: Design Complete — Awaiting Implementation Approval*
