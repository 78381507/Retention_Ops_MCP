# /artifacts/example_brief_critical.md
# Retention Alert — CRITICAL
Date: 2026-01-19

## Situation
AT_RISK% increased sharply vs baseline.
- Current AT_RISK%: 27.1
- Baseline (7-day avg): 21.5
- Delta: +5.6 pp
- Relative delta: +26.0%
- Sample size: 50,842 customers
- Alert flag: TRUE

## Severity
CRITICAL (threshold: +25% relative increase)

## What is ready (no manual analysis required)
Priority audiences generated from existing SQL outputs:
- P1 (Prevention): 104 customers (ACTIVE + HIGH risk)
- P2 (Intervention): 468 customers (AT_RISK + HIGH/MED risk)
- P3 (Win-back): 251 customers (INACTIVE + HIGH risk, recent focus if configured)

## Immediate playbook (CRITICAL)
1) CRM/Marketing (today): launch P1 outreach (high-value active customers).
2) CRM/Marketing (today): launch P2 win-back sequence (batch campaign).
3) CRM/Marketing (this week): run P3 win-back (stronger incentive + survey).
4) Operations: escalate incident ownership and confirm downstream systems:
   - data freshness (orders ingested as expected)
   - baseline validity (7-day history present)
   - no abnormal spikes due to data quality issues

## Artifacts
- Dashboard (Looker): [LINK_PLACEHOLDER_LOOKER_DASHBOARD]
- Export P1 (Google Sheet): https://docs.google.com/spreadsheets/d/EXAMPLE_P1_20260119
- Export P2 (Google Sheet): https://docs.google.com/spreadsheets/d/EXAMPLE_P2_20260119
- Export P3 (Google Sheet): https://docs.google.com/spreadsheets/d/EXAMPLE_P3_20260119
- Ticket: RET-2412 (priority: URGENT)

## Notes / Guardrails
- Single alert metric only (AT_RISK% deviation) to avoid alert fatigue.
- Anti-noise gates passed (sample size and baseline warmup).
- Safe rollback: disable MCP layer; SQL detection continues unchanged.
