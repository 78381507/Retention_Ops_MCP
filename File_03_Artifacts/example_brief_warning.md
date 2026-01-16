# Retention Alert — WARNING
Date: 2026-01-12

## Situation
AT_RISK% increased vs baseline.
- Current AT_RISK%: 25.2
- Baseline (7-day avg): 21.4
- Delta: +3.8 pp
- Relative delta: +18.0%
- Sample size: 49,378 customers
- Alert flag: TRUE

## What is ready (no manual analysis required)
Priority audiences generated from existing SQL outputs:
- P1 (Prevention): 87 customers  (ACTIVE + HIGH risk)
- P2 (Intervention): 312 customers (AT_RISK + HIGH/MED risk)
- P3 (Win-back): not exported for WARNING (CRITICAL only)

## Immediate playbook (WARNING)
1) CRM/Marketing: launch P1 outreach (small batch, high ROI).
2) CRM/Marketing: prepare P2 win-back campaign (sequence + incentive if applicable).
3) Data/Operations: confirm no data quality anomalies (sample size stable, baseline valid).

## Artifacts
- Dashboard (Looker): [LINK_PLACEHOLDER_LOOKER_DASHBOARD]
- Export P1 (Google Sheet): https://docs.google.com/spreadsheets/d/EXAMPLE_P1_20260112
- Export P2 (Google Sheet): https://docs.google.com/spreadsheets/d/EXAMPLE_P2_20260112
- Ticket: RET-2401 (created by playbook)

## Notes / Guardrails
- Single alert metric only (AT_RISK% deviation) to avoid alert fatigue.
- Baseline warmup required: 7 full days (met).
- System is safe to disable: MCP can be turned off without impacting SQL pipelines.
