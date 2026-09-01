# PRODUCTION ANALYTICS ARCHITECTURE DESIGN

## System Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    COLLECTIONS ANALYTICS PLATFORM               │
│                    Raw → Staging → Clean → Golden → Metrics     │
└─────────────────────────────────────────────────────────────────┘

LAYER 1: DATA INGESTION (Real-time + Batch)
═══════════════════════════════════════════════════════════════════

                    ┌──────────────────────────┐
                    │   Source Systems         │
                    ├──────────────────────────┤
    ┌──────────────►│ • Core Banking (Daily)   │
    │               │ • Telephony Vendors      │
    │               │ • WhatsApp/SMS Gateways  │
    │               │ • Field Tracking App     │
    │               │ • Payment Provider APIs  │
    │               └──────────────────────────┘
    │
    │  API / Database connectors
    │  (scheduled every 6 hours + event-driven)
    │
    ▼
┌───────────────────────────────────────────────────────────────────┐
│  STAGING LAYER (Raw ingestion)                                    │
│  ───────────────────────────────────────────────────────────────  │
│  stg_payments_raw (25.5k rows)                                   │
│  stg_calls_raw (91.3k rows)                                      │
│  stg_call_dispositions_raw (35k rows)                            │
│  stg_sms_events_raw (45k rows)                                   │
│  stg_whatsapp_events_raw (60.6k rows)                            │
│  stg_field_visits_raw (25k rows)                                 │
│  stg_promises_to_pay_raw (18k rows)                              │
│  stg_agent_sessions_raw (15k rows)                               │
│                                                                   │
│  ✓ Schema validation (detect schema changes)                     │
│  ✓ Format normalization (date/time parsing)                      │
│  ✓ Null value handling (log missing values)                      │
│  ✓ Deduplication tagging (mark duplicates, don't drop yet)      │
│  ✓ Late-arrival detection (flag >7 day delays)                   │
│  ✓ Schema versioning (track source system changes)               │
└───────────────────────────────────────────────────────────────────┘
    │
    │  Data Quality Checks:
    │  • Null rate < 5% per field
    │  • Duplicates reported to data steward
    │  • Timezone validation
    │  • Date range validation (no future dates)
    │
    ▼
┌───────────────────────────────────────────────────────────────────┐
│  TRANSFORMATION LAYER (Cleaning & normalization)                  │
│  ───────────────────────────────────────────────────────────────  │
│  trm_payments (payment deduplication, failed payment removal)     │
│  trm_calls_normalized (timezone standardization to UTC)          │
│  trm_dispositions_cleaned (multi-disposition handling)           │
│  trm_events_unified (voice, SMS, WhatsApp, field into one log)   │
│  trm_ptp_validated (cross-check with actual payments)            │
│  trm_agent_mapped (employee_code as canonical ID)                │
│  trm_campaign_versioned (version each campaign_id)               │
│                                                                   │
│  Logic Applied:                                                  │
│  • Remove 600 duplicate/failed payments                          │
│  • Normalize 8000 misclassified timezones                        │
│  • Resolve 2400 multi-disposition calls (keep first)             │
│  • Validate 1400 unmatched PTPs                                  │
│  • Map 50 agent ID conflicts to employee_code                    │
│  • Version 3 campaign redefinitions                              │
│                                                                   │
│  Audit Trail:                                                    │
│  • trm_data_quality_flags (why records removed/modified)         │
│  • trm_transformation_log (what changed, when, by whom)          │
│  • trm_lineage_metadata (track data lineage)                     │
└───────────────────────────────────────────────────────────────────┘
    │
    │  Incremental Processing:
    │  • New payments processed daily
    │  • Late-arriving events identified
    │  • Backfill logic for corrections
    │
    ▼
┌───────────────────────────────────────────────────────────────────┐
│  GOLDEN LAYER (Source of Truth)                                   │
│  ───────────────────────────────────────────────────────────────  │
│  dim_accounts (account master, deduplicated)                      │
│  dim_borrowers (borrower master, deduplicated)                    │
│  dim_agents (agent master, standardized IDs)                      │
│  dim_campaigns (campaign master, versioned)                       │
│  fact_recovery (24.9k clean payment records)                      │
│  fact_interactions (unified customer event log)                   │
│  fact_promises_to_pay (validated PTPs with payment status)        │
│  fact_field_visits (collections field outcomes)                   │
│                                                                   │
│  ✓ Primary Keys: recovery_id, account_id, borrower_id            │
│  ✓ Foreign Keys: All IDs point to corresponding dimensions        │
│  ✓ Data Contracts:                                               │
│    - fact_recovery.amount > 0 (no negative/zero amounts)         │
│    - fact_recovery.amount < 500k (outlier check)                 │
│    - fact_recovery.event_at_utc is NOT NULL                      │
│    - fact_recovery is deduplicated on payment_reference          │
│  ✓ Quality Flags: is_retry, is_outlier, data_quality_score       │
│  ✓ Completeness: 98.7% (after cleaning)                          │
│                                                                   │
│  Refresh Cadence:                                                │
│  • Incremental: daily (new payments)                             │
│  • Full rebuild: weekly (catch corrections)                      │
│  • Late-arrival window: 30 days                                  │
└───────────────────────────────────────────────────────────────────┘
    │
    │  Data Governance:
    │  • Ownership: Data Ops team
    │  • Change log: All transformations audited
    │  • SLA: 99.5% availability
    │
    ▼
┌───────────────────────────────────────────────────────────────────┐
│  FEATURE ENGINEERING LAYER (Business logic)                       │
│  ───────────────────────────────────────────────────────────────  │
│  fea_monthly_metrics (aggregated by month)                        │
│  fea_cohort_recovery (by account vintage, normalized)             │
│  fea_channel_performance (last-touch attribution)                 │
│  fea_agent_productivity (tenure, session analysis)                │
│  fea_campaign_metrics (RPC, PTP rate, conversion)                 │
│  fea_portfolio_mix (DPD, risk segment, loan type distribution)    │
│                                                                   │
│  Features Built:                                                 │
│  • recovery_amount_usd                                           │
│  • recovery_month                                                │
│  • cohort_quarter (when account opened)                          │
│  • attributed_channel (last touchpoint before payment)           │
│  • agent_tenure_days (standardized)                              │
│  • days_since_ptp (if PTP was made)                              │
│  • dpd_at_payment (DPD when payment occurred)                    │
│  • risk_segment_at_payment                                       │
│                                                                   │
│  Validations:                                                    │
│  • No NULLs in key business features                             │
│  • distributions match expected ranges                           │
│  • temporal consistency (dates monotonic)                        │
└───────────────────────────────────────────────────────────────────┘
    │
    ▼
┌───────────────────────────────────────────────────────────────────┐
│  METRICS LAYER (Business KPIs)                                    │
│  ───────────────────────────────────────────────────────────────  │
│  mtc_monthly_recovery                                             │
│  ├─ Total Recovery Amount (₹)                                    │
│  ├─ Number of Recoveries (#)                                     │
│  ├─ Recovery Rate (% of accounts)                                │
│  └─ MoM Growth Rate (%)                                          │
│                                                                   │
│  mtc_channel_metrics                                             │
│  ├─ Channel (VOICE, SMS, WHATSAPP, FIELD)                        │
│  ├─ Recovery $ attributed to channel                             │
│  ├─ Conversion rate (interactions → payment)                     │
│  └─ Cost per ₹ recovered                                         │
│                                                                   │
│  mtc_agent_productivity                                          │
│  ├─ Calls handled                                                │
│  ├─ RPC rate (% answered)                                        │
│  ├─ Conversion rate (call → payment)                             │
│  ├─ Avg recovery per agent-hour                                  │
│  └─ Tenure effect on productivity                                │
│                                                                   │
│  mtc_campaign_metrics                                            │
│  ├─ Campaign name & version                                      │
│  ├─ Target definition (DPD >= X)                                 │
│  ├─ Recovery amount & rate                                       │
│  ├─ RPC, PTP rate, conversion                                    │
│  └─ Cost per ₹ recovered                                         │
│                                                                   │
│  mtc_cohort_recovery (normalized by account age)                 │
│  ├─ Cohort (when account opened)                                 │
│  ├─ Recovery rate trend (month 1 to month N)                     │
│  └─ Survivorship analysis                                        │
│                                                                   │
│  Metric Definitions:                                             │
│  • RPC = ANSWERED calls / TOTAL calls (%)                        │
│  • PTP kept rate = KEPT promises with payment / total KEPT (%)  │
│  • Recovery rate = Accounts with payment / total accounts (%)    │
│  • Conversion = Accounts with payment after interaction (%)      │
│  • Cost per ₹ = Campaign spend / recovery amount                 │
└───────────────────────────────────────────────────────────────────┘
    │
    ▼
┌───────────────────────────────────────────────────────────────────┐
│  PRESENTATION LAYER (BI & Dashboards)                             │
│  ───────────────────────────────────────────────────────────────  │
│                                                                   │
│  Dashboard 1: Executive Summary (60-second overview)             │
│  ├─ KPI cards: Total recovery, MoM growth, accounts paid         │
│  ├─ Trend: Recovery amount by month (12 months)                  │
│  ├─ Channel breakdown: Which channel driving recovery            │
│  └─ Risk: Top risks (duplicates, anomalies, missed SLAs)         │
│                                                                   │
│  Dashboard 2: Deep Dive (Campaign Performance)                   │
│  ├─ Campaign table: name, DPD target, recovery, RPC, PTP rate   │
│  ├─ Channel comparison: Voice vs SMS vs Field                    │
│  ├─ Trend analysis: Is each channel improving or declining?      │
│  └─ Agent ranking: Top 10 agents by recovery per hour            │
│                                                                   │
│  Dashboard 3: Cohort Analysis (Portfolio normalization)          │
│  ├─ Cohort recovery rate: Comparing same accounts over time      │
│  ├─ No portfolio mix bias, see true operational improvement      │
│  ├─ Survivorship tracking: Accounts written off mid-period       │
│  └─ DPD progression: How accounts are resolving                  │
│                                                                   │
│  Dashboard 4: Data Quality (Internal operations)                 │
│  ├─ Duplicate detection: How many duplicate payments found       │
│  ├─ Latency: How fresh is the data?                              │
│  ├─ Completeness: % null by field                                │
│  └─ Anomalies: Unusual patterns flagged                          │
│                                                                   │
│  Alerting:                                                       │
│  • If MoM growth > 8%: Investigate for data issues               │
│  • If duplicate rate > 2%: Page on-call data engineer            │
│  • If latency > 7 days: Alert data platform team                 │
│  • If metric null rate > 3%: Escalate to stakeholders            │
└───────────────────────────────────────────────────────────────────┘


LAYER 2: DATA CONTRACTS & QUALITY GATES
═══════════════════════════════════════════════════════════════════

┌─────────────────────────────────────────────────────┐
│ Staging → Transformation Gate                       │
├─────────────────────────────────────────────────────┤
│ ✓ Check 1: No future dates                         │
│ ✓ Check 2: Negative amounts < 0.1%                 │
│ ✓ Check 3: Null rate < 5% per field                │
│ ✓ Check 4: Schema matches expected columns         │
│ ✓ Check 5: Row count within 10% of yesterday       │
│ ✓ Check 6: Primary key uniqueness                  │
│ Action: Block transformation if checks fail        │
└─────────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────┐
│ Transformation → Golden Gate                        │
├─────────────────────────────────────────────────────┤
│ ✓ Check 1: All payments deduplicated               │
│ ✓ Check 2: Foreign keys point to valid dims        │
│ ✓ Check 3: No NULL in critical fields              │
│ ✓ Check 4: Date ranges are sensible                │
│ ✓ Check 5: Outlier detection (z-score > 3)         │
│ ✓ Check 6: Audit trail complete                    │
│ Action: Flag anomalies, don't block (analysts     │
│         can investigate)                           │
└─────────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────┐
│ Golden → Metrics Gate                              │
├─────────────────────────────────────────────────────┤
│ ✓ Check 1: Metric definitions match spec           │
│ ✓ Check 2: Metrics sum to expected range           │
│ ✓ Check 3: YoY consistency (if available)          │
│ ✓ Check 4: No metric is NULL                       │
│ ✓ Check 5: Refresh completed within SLA (1 hour)  │
│ Action: Quarantine metrics if checks fail          │
└─────────────────────────────────────────────────────┘


LAYER 3: LATE-ARRIVING DATA & BACKFILL LOGIC
═══════════════════════════════════════════════════════════════════

Timeline:
  Day 1: Payment occurs
  Day 1 evening: Ingested into staging
  Day 2: Transformed into golden
  Day 2: Metrics calculated
  Day 3-7: "Late-arrival window" - payment might correct/backfill

Strategy:
  • Keep 30-day correction window
  • Store both "latest" and "as-of-date" versions
  • When backfill occurs:
    1. Reprocess last 30 days of payments
    2. Recalculate affected monthly metrics
    3. Log correction in audit table
    4. Notify analysts if significant change (>5%)

Example:
  On Aug 15: Report shows Jul recovery = ₹100M
  On Aug 20: Late payment discovered (was mislabeled as Aug)
  On Aug 21: Reprocessed... Jul recovery = ₹101M (recalculated)
  Notification: "Jul recovery revised +1% due to late arrival"


LAYER 4: INCREMENTAL PROCESSING STRATEGY
═══════════════════════════════════════════════════════════════════

Daily Incremental Load:
  1. stg_payments_raw: Get all payments from last 2 days
  2. Deduplicate against trm_payments
  3. New payments → add to fact_recovery
  4. Recalculate metrics for last 2 months (accounts for backfill)
  5. Validate against data contracts
  6. Update presentation layer
  
  Duration: ~15 minutes
  Cost: ~₹50/day (incremental compute)

Weekly Full Rebuild:
  1. Drop and recreate all transformation tables
  2. Reprocess all source data from scratch
  3. Recalculate all metrics
  4. Compare to incremental results (should match within 0.1%)
  5. If mismatch > 0.1%, investigate
  
  Duration: ~2 hours
  Cost: ~₹200/week (full compute)
  
  Why: Catch bugs, schema changes, late-arriving data


LAYER 5: MONITORING & ANOMALY DETECTION
═══════════════════════════════════════════════════════════════════

Real-time Alerts:
  • If duplicate payment rate > 2%: Alert data engineer
  • If record count drops > 20%: Alert data steward
  • If transformation latency > 30 min: Page on-call

Daily Monitoring:
  • Data quality dashboard: See null rates, duplicates, schema issues
  • Metric drift: Compare yesterday to 30-day average
  • If metric down > 10%: Investigate before publishing

Weekly Review:
  • Anomaly report: Any unusual patterns in data
  • Backfill summary: How many corrections occurred
  • Schema changes: Any new fields or field removals
  • Performance: Data latency and processing time trends


LAYER 6: IMPLEMENTATION ROADMAP
═══════════════════════════════════════════════════════════════════

Week 1-2: Setup & Architecture
  □ Create staging tables in data warehouse
  □ Build ETL pipelines (Apache Airflow or equivalent)
  □ Implement data quality checks

Week 3-4: Data Cleaning
  □ Implement deduplication logic
  □ Timezone normalization
  □ Agent ID mapping

Week 5-6: Transformation Layer
  □ Build fact tables (payments, interactions)
  □ Implement feature engineering
  □ Create lineage tracking

Week 7-8: Metrics & Dashboards
  □ Define all metric calculations
  □ Build executive dashboard
  □ Setup alerting

Week 9-10: Testing & Validation
  □ Reconcile metrics with legacy system
  □ Data quality testing
  □ Performance optimization

Week 11-12: Go-live & Operations
  □ Production deployment
  □ Documentation & training
  □ Handoff to operations team


LAYER 7: COST ESTIMATION
═══════════════════════════════════════════════════════════════════

Infrastructure (Annual):
  • Data warehouse: ₹15-20 Lakhs (cloud storage + compute)
  • BI tool license: ₹5-8 Lakhs (Tableau or Looker)
  • ETL platform: ₹5-10 Lakhs (Airflow, DBT, or managed service)
  • Total: ₹25-38 Lakhs/year

Personnel (Annual):
  • Data engineer (build pipeline): ₹15-25 Lakhs
  • Analytics engineer (metrics): ₹12-18 Lakhs
  • Data analyst (dashboards): ₹10-15 Lakhs
  • Data steward (governance): ₹8-12 Lakhs
  • Total: ₹45-70 Lakhs/year

Total Annual Cost: ₹70-108 Lakhs
ROI: Saves ~₹5-10 Cr/year in operational efficiency (no more manual reporting)
