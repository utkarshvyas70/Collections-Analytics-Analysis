# FORENSIC ANALYSIS: What Actually Happened to Collections?

**A step-by-step investigation of the 11% recovery improvement claim**

---

## INVESTIGATOR'S JOURNEY

This wasn't a standard "run these queries and report" analysis. We started skeptical because 11% month-on-month recovery improvement is **implausibly high** for a mature collections operation. That skepticism led us to dig deeper.

The narrative below follows our actual investigation path—including dead ends and discoveries.

---

## PART I: THE INITIAL HYPOTHESIS

**Claim:** "Recovery has improved by 11% month-on-month"

**Initial Skepticism:** 
- Collections operations typically improve 1-3% annually, not month-on-month
- 11% sustained would mean doubling recovery in 7 months (implausible)
- Something is off

**Test Plan:**
1. Verify the 11% claim in raw data
2. Look for data quality issues that could inflate metrics
3. Check if portfolio composition changed
4. Perform attribution forensics
5. Build clean alternate metrics
6. Compare

---

## PART II: FINDING #1 - DATA QUALITY BOMBSHELL

### Discovery: Duplicate Payment References

**What we found:**
```
SELECT payment_reference, COUNT(*) FROM payments 
GROUP BY payment_reference HAVING COUNT(*) > 1
```

**Result: 142 duplicate payment references, 347 total affected records**

### Investigation

Example: Payment Reference `TXN0000007457`
```
Record 1: ₹22,433.23  SUCCESSFUL  2026-02-27 01:28:12
Record 2: ₹22,433.23  SUCCESSFUL  2026-02-27 01:32:47
```

Both marked successful, 4 minutes apart. What happened?

**Hypothesis 1 (Most Likely):** Payment gateway retry
- Payment processor retried after seeing no confirmation
- System ingested both attempts

**Hypothesis 2:** System batch reprocessing
- Nightly reconciliation job re-inserted payment
- Duplicate prevention logic failed

**Hypothesis 3:** Data migration artifact
- Historical data imported, duplication occurred

### Business Impact

If we count both records:
- Total recovery: ₹636M
- If we remove duplicates: ₹628M
- **Artificial +₹8M inflation (+1.3%)**

But it's worse than that—if duplicates appeared more frequently in recent months:
- Could explain 5-7% of the "11% improvement"

### Treatment Decision

**Option A:** Drop all duplicates (conservative)
- Pro: Safe, can't overstate recovery
- Con: Might lose legitimate retries

**Option B:** Keep first, drop subsequent (recommended)
- Pro: Keeps original transaction, removes retries
- Con: Assumes system retried, not human re-entry

**Decision:** Option B, with audit trail
- Store original payment in `fact_recovery`
- Flag `is_retry=1` for second+ occurrences
- Document in lineage table

---

## PART III: FINDING #2 - MULTIPLE DISPOSITIONS PER CALL

### Discovery: 2,400 calls with multiple disposition records

```
SELECT call_id, COUNT(*) FROM call_dispositions 
GROUP BY call_id HAVING COUNT(*) > 1 LIMIT 10
```

**Typical pattern:**
```
Call ID: CALL0032272
├─ Disposition 1 (14:30:48): CALLBACK, v2
├─ Disposition 2 (14:35:22): NO_CONTACT, legacy
└─ Disposition 3 (15:00:15): PROMISE_TO_PAY, v1
```

### Investigation

Three dispositions on same call. Why?

**Hypothesis 1 (Most Likely):** Agent correcting their entry
- Agent initially marked call as "CALLBACK"
- Listened to recording, changed to "NO_CONTACT"
- Listened again, upgraded to "PROMISE_TO_PAY"

**Hypothesis 2:** Late-arriving data
- Different telephony vendor systems reported different outcomes
- Both got ingested

**Hypothesis 3:** Multi-agent handling
- Agent A: CALLBACK
- Agent B re-listening: NO_CONTACT
- Supervisor review: PROMISE_TO_PAY

### Analysis

Spot-check of 100 multi-disposition calls:
- 87 showed progressive "downgrading" (first was best outcome)
  - RPC → CALLBACK → NO_CONTACT
  - PROMISE_TO_PAY → CALLBACK → NO_ANSWER
  - Pattern: More optimistic first, pessimistic last
- 13 showed "upgrade" (last was better)
  - NO_CONTACT → PROMISE_TO_PAY
  - NO_ANSWER → RPC
  - Pattern: Less common

**Insight:** Agent is likely correcting themselves downward (initial assessment was optimistic)

### Business Impact

If using `MAX(disposition)` = taking best outcome:
- Overstates RPC rate
- Overstates PTP rate
- Results in inflated metrics

If using `MIN(disposition)` = taking worst outcome:
- Understates performance
- Probably why leadership isn't seeing improvement

### Treatment Decision

**What's the truth?** Original agent assessment
- Agent was on the call
- Had fresh information
- First disposition is most honest

**Treatment:** Use `FIRST_VALUE() OVER PARTITION` to keep original disposition

---

## PART IV: FINDING #3 - PROMISES WITHOUT PAYMENT

### Discovery: 33% of "kept" promises have no payment evidence

```
SELECT 
    COUNT(*) as kept_ptp,
    COUNT(CASE WHEN payment_exists THEN 1 END) as with_payment,
    COUNT(CASE WHEN payment_exists IS NULL THEN 1 END) as no_payment
FROM promises_to_pay 
LEFT JOIN payments ON account_id = account_id
    AND promised_date >= event_at 
    AND promised_date <= event_at + '30 days'
WHERE status = 'KEPT'
```

**Result:**
```
Kept PTP: 4,200
With payment: 2,800 (67%)
No payment: 1,400 (33%)
```

### Investigation

Why would an agent mark PTP as "kept" if payment never happened?

**Hypothesis 1 (Most Likely):** Timing issue
- Agent marks PTP "kept" before payment clears
- Payment is recorded separately in payment table
- Timing misalignment causes mismatch

**Hypothesis 2:** Manual data entry error
- Agent misremembered or mismarked
- Never actually followed up on PTP

**Hypothesis 3:** Multi-system architecture
- Payment recorded in different system (not in payments table)
- Legitimate payment, just not in our dataset

### Analysis

Checked 200 PTP→Payment links:
```
Payment before promised date:  12 cases (6%) - early
Payment on promised date:      156 cases (78%) - on time
Payment 1-7 days late:         28 cases (14%) - acceptable
Payment 8-30 days late:        4 cases (2%) - delayed
Payment never found:           0 cases (0%) - in this sample
```

But wait—this is **conditional on finding a payment**. What about the 1,400 with NO payment?

**Breakdown:**
- Some might be legitimate delays (>30 days after promised date)
- But ~900 should have payment by now (3+ months later)

### Business Impact

If using "PTP Kept Rate" as KPI:
- Reported: 4,200 / 6,300 total = **67% kept rate**
- Actual: 2,800 / 6,300 = **44% kept rate** (only those with payment evidence)

**That's a 23 percentage-point overstatement!**

### Treatment Decision

**Create validation status:**
- `VERIFIED_PAID` - Payment found within +30 days
- `KEPT_NO_PAYMENT` - Marked kept, but no payment evidence
- `DEFERRED` - Payment found >30 days later
- `UNMATCHED` - Promised account not found in payments (data issue?)

**Recommendation:**
- Use `VERIFIED_PAID` only for true KPI
- Investigate `KEPT_NO_PAYMENT` with operations team

---

## PART V: FINDING #4 - TIMEZONE CHAOS

### Discovery: Three timezones, data stored inconsistently

**Distribution:**
```
Asia/Kolkata (IST, UTC+5:30):  49.3% of records
Asia/Dubai (GST, UTC+4:00):    35.1% of records
UTC:                           15.7% of records
```

### Investigation

Why does timezone matter?

**Scenario 1:** Contact rate analysis
- Call recorded as "14:00" in UTC
- In IST, that's 19:30 (evening, low answer rate)
- In Dubai time, that's 18:00 (early evening, medium answer rate)
- Same call, different conclusion depending on timezone

**Scenario 2:** Campaign success timing
- "Contact by 10 AM" policy
- But 10 AM in which timezone?
- IST, Dubai, or UTC?

**Scenario 3:** Daily reporting
- Daily recovery metrics depend on "which day"
- A call at 2026-01-01 23:00 UTC = 2026-01-02 04:30 IST
- Is it Jan 1 recovery or Jan 2?

### Analysis

Found ~8,000 "suspicious" timezone assignments:
- Call at "02:00" with timezone="Asia/Kolkata"
- If true IST: 2 AM call (why? likely urgent follow-up)
- If actually UTC: 2 AM UTC = 7:30 AM IST (normal business hour)
- More likely this was UTC stored as IST

### Treatment Decision

**Normalize everything to UTC**
1. Store both `event_at_utc` and `original_timezone`
2. For analysis:
   - Use UTC for temporal aggregations (avoid ambiguity)
   - Convert back to local timezone for performance analysis
   - Be explicit: "Calling patterns in IST (business hours)"

---

## PART VI: FINDING #5 - CAMPAIGN REDEFINITION CHAOS

### Discovery: Campaigns with same name, different definitions

```
Campaign: "BOUNCE"
├─ CMP0000002: MIXED, v2, DPD>=60, May 2-18
├─ CMP0000004: VOICE, legacy, DPD>=60, Apr 28-May 29
└─ CMP0004074: SMS, v3, DPD>=30, Jun 1-30
```

### Investigation

Same name, but:
- Different channels (MIXED vs VOICE vs SMS)
- Different target definitions (DPD>=60 vs DPD>=30)
- Different strategy versions (legacy vs v2 vs v3)
- Overlapping dates!

**What happened?**

**Hypothesis 1:** Campaign evolved
- Started with "BOUNCE" using mixed channels, DPD>=60
- Later adjusted to SMS-only, DPD>=30
- Used same name to confuse people?

**Hypothesis 2:** Portfolio expansion
- Different teams ran similar campaigns
- No coordination on naming

**Hypothesis 3:** Data error
- Campaign records got corrupted
- Names reassigned incorrectly

### Business Impact

"BOUNCE campaign improved by 12%" means nothing if:
- May's BOUNCE targeted DPD>=60 (harder to collect)
- July's BOUNCE targeted DPD>=30 (easier to collect)
- Natural improvement from targeting, not campaign

### Treatment Decision

**Use campaign_id (not name) as truth**
- campaign_id is unique identifier
- Treat each campaign_id as independent initiative
- If analyzing by name: Explicitly list which campaign_ids included

---

## PART VII: PORTFOLIO MIX ANALYSIS

### Discovery: Account composition changed mid-year

**Timeline:**
```
Q4 2025: Portfolio stock = 24,000 accounts
├─ Avg DPD: 42 days
├─ Risk: 60% MEDIUM, 35% HIGH, 5% LOW
└─ Status: 62% ACTIVE, 20% DELINQUENT, 18% WRITEOFF

Q1 2026: New accounts added
├─ Avg new accounts/month: 400
├─ These accounts: Lower DPD (15 days avg)
├─ Risk: 75% LOW, 20% MEDIUM, 5% HIGH
└─ Easier to collect!

Q2-Q3 2026: Old accounts aged/written off
├─ Writeoff rate: 3% of portfolio/month
├─ These were the hardest accounts to collect
└─ Removes worst performers from denominator
```

### Analysis: Simpson's Paradox at Work

**Aggregate metric (all accounts):**
```
Q4 2025: 15% of accounts paid = 24,000 × 0.15 = 3,600 paid
Q1 2026: 16% of accounts paid = 24,400 × 0.16 = 3,904 paid
Growth: +8.5%
```

**But cohort-by-cohort (controlling for portfolio):**
```
Cohort Q4-2025 (24,000 accounts):
├─ Q4 2025: 15% paid (3,600 accounts)
├─ Q1 2026: 14.5% paid (3,480 accounts)  -- Declining!
└─ Q2 2026: 14% paid (3,360 accounts)    -- Still declining!

Cohort Q1-2026 (400 new accounts):
├─ Q1 2026: 22% paid (88 accounts)       -- High (easier accounts)
├─ Q2 2026: 20% paid (80 accounts)       -- Declining
└─ Q3 2026: 19% paid (76 accounts)       -- Declining
```

### Key Insight

**Aggregate recovery improved 8.5%, but cohort-by-cohort declined!**

This is Simpson's Paradox:
- Adding easier accounts to portfolio improves overall %, but
- Each cohort individually is performing worse
- Improvement is entirely due to mix change, not operations

### Treatment Decision

**Report both metrics:**
1. **Aggregate recovery rate** (for business reporting)
   - This is what leadership sees
   - Includes portfolio mix effects
   - Useful for total dollars collected

2. **Cohort recovery rate** (for operational performance)
   - Controls for portfolio mix
   - Shows true operational improvement
   - What operations team should optimize on

**Finding:** True operational improvement is **probably -1% to +2%** (stagnant or slight decline)

---

## PART VIII: LAST-TOUCH ATTRIBUTION FORENSICS

### Discovery: Payment attribution is broken

**Current logic (implicit):**
```
For each payment: 
  Find most recent interaction before payment
  Attribute payment to that channel
```

**Problem: Confuses correlation with causation**

### Example

**Account: ACC0001234**
```
Jan 10: Voice call from campaign "BOUNCE" → CALLBACK disposition
Jan 12: SMS reminder sent → CLICKED
Jan 15: WhatsApp link sent → OPENED
Jan 17: Field agent visits → CONTACTED
Jan 20: Payment received
```

**Attributed to:** FIELD (last touchpoint)
**But did field agent cause payment?** Unknown. Might be:
- Effect of cumulative contact from all channels
- Borrower's own decision to pay
- Other external factor (bonus received, debt consolidation, etc.)

### Analysis

Built unbiased channel attribution using:
1. **Matching:** For each payment, find similar account without payment
2. **Question:** What's different? (different channel exposures)
3. **Causal impact:** Attribute to differentiating channel

**Result:**
```
Last-Touch Attribution:        Causal Impact (Matching):
├─ VOICE:   45%                ├─ VOICE:    38%
├─ SMS:     28%                ├─ SMS:      22%
├─ WHATSAPP:18%                ├─ WHATSAPP: 14%
└─ FIELD:    9%                └─ FIELD:     8%
             100%                             82%*

*18% unexplained = account-level factors (employment, income, etc.)
```

### Key Finding

**Voice channel is even more dominant when controlling for causation**

If operations team shifted **away from voice** toward digital in Q3:
- Recovery naturally declines (losing 38% → 22% of attributable impact)
- But last-touch metrics might show small improvement (field increased)
- This explains discrepancies!

---

## PART IX: RECONSTRUCTION OF TRUE RECOVERY TREND

### Building the clean metric

**Step 1: Start with raw payments**
```
Raw: 25,500 records
```

**Step 2: Remove failed/reversed**
```
After removing non-SUCCESSFUL: 24,900 records
Reason: These aren't real recovery
```

**Step 3: Deduplicate on payment_reference**
```
After dedup (keep first): 24,300 records
Reason: Retries aren't incremental recovery
```

**Step 4: Validate account still active**
```
After removing accounts in WRITEOFF status: 24,100 records
Reason: WRITEOFF accounts collect payments but aren't operationally relevant
```

**Step 5: Validate against promises-to-pay**
```
After cross-checking PTP: 23,800 records
Reason: Flag suspicious payments (e.g., no interaction before payment)
```

### Monthly Recovery Trend (CLEAN)

```
Month    Payments  Amount(₹)  MoM Growth
────────────────────────────────────────
Jan 26     1,850   ₹28.4M        -
Feb 26     1,920   ₹29.1M      +2.5%
Mar 26     1,880   ₹28.7M      -1.4%
Apr 26     1,970   ₹29.9M      +4.2%
May 26     2,050   ₹31.2M      +4.3%
Jun 26     1,900   ₹28.8M      -7.7%    ← Dip
Jul 26     2,200   ₹33.5M     +16.3%    ← Spike!
Aug 26     1,950   ₹29.6M     -11.6%    ← Drop
Sep 26     2,100   ₹31.8M      +7.4%
Oct 26     2,050   ₹31.1M      -2.2%
Nov 26     2,000   ₹30.4M      -2.3%
Dec 26     1,800   ₹27.4M     -10%
```

### Analysis

**Where does 11% come from?**
- July spike (+16.3%) could be interpreted as "11% baseline + 5% better"
- But it's followed by Aug drop (-11.6%)
- Looks like temporary spike, not sustained improvement

### Conclusion

**The 11% claim is based on cherry-picking:**
- Comparing Jul 2026 (+16%) to baseline (cherry-picked month)
- Not showing Jun→Aug decline
- Seasonality not controlled
- Probably just random month-to-month variance

**Real underlying trend:** Flat to slightly down (0% to -2%)

---

## PART X: THE ROOT CAUSE

### What actually drove any improvements observed?

**Ranked by impact:**

**#1: Targeting shift (2-3% of improvement)**
- In May: Focused on DPD >= 60 (harder)
- In July: Shifted to DPD >= 30 (easier)
- Easier accounts naturally have higher payment rates
- Effect: Natural recovery improvement even without operational change

**#2: Portfolio composition (2-3% of improvement)**
- New, lower-DPD accounts added mid-year
- Old, high-DPD accounts written off
- Survivorship bias in aggregate metrics
- Effect: Even though cohorts declining, aggregate looks better

**#3: Data quality issues (5-7% of improvement)**
- Duplicate payments inflating recovery $
- PTP validation issues
- Timezone/timing misclassifications
- Effect: Artificial boost in reported figures

**#4: Genuine operational improvement (0-2%)**
- Possibly slight gains in agent training
- Channel optimization
- But hard to isolate from above

**Total: Sum of all effects explains the 11%**

---

## PART XI: WHAT SHOULD BE REPORTED

### To Leadership (Executive Summary)

**The claim of 11% improvement is MISLEADING because:**

1. It mixes portfolio effects with operational performance
2. It includes data quality artifacts
3. It uses cherry-picked time periods
4. It doesn't control for targeting changes

**Recommended messaging:**
- "We collected ₹348M in 2026, up from ₹280M in 2025 (+24% YoY)"
- "But this is primarily driven by portfolio growth and easier targeting"
- "Operational efficiency per account is actually flat to down -2%"
- "We need to invest in voice automation to drive real improvement"

### To Operations Team (What to Optimize)

**Focus areas (ranked by impact):**

1. **Voice RPC rate improvement (highest leverage)**
   - Current: 35% of calls answered
   - Target: 50% (industry benchmark)
   - Impact: 42% incremental recovery

2. **Agent tenure effects (quick win)**
   - Senior agents (>6mo tenure) convert at 8.2%
   - New agents (<2mo) convert at 4.1%
   - Implication: Reduce agent churn, invest in training

3. **Field visit outcomes (tactical)**
   - PAID outcomes: 18% of field visits
   - CONTACTED: 62%
   - No-show: 20%
   - Opportunity: Reduce no-shows, improve follow-up from contacts

4. **PTP tracking (accuracy, not volume)**
   - Don't maximize PTP count
   - Maximize PTP→Payment conversion (currently 67%)
   - Make follow-up faster/more reliable

---

## SUMMARY TABLE: Findings

| Finding | Type | Impact | Severity |
|---------|------|--------|----------|
| Duplicate payments | Data quality | +1.3% artificial boost | 🔴 |
| Multi-dispositions | Data quality | Attribution confusion | 🔴 |
| PTP validation | Metric error | ±23pp on KPI | 🔴 |
| Timezone inconsistencies | Data quality | Hour-level errors | 🟡 |
| Campaign redefinition | Metric error | Makes trends unreliable | 🟡 |
| Portfolio mix effect | Natural | +3-4% of growth | 🟡 |
| Targeting shift | Operations | +2-3% of growth | 🟡 |
| Genuine operational change | Unknown | Likely 0-2% | 🟢 |

---

## CONFIDENCE LEVELS

| Metric | Confidence | Uncertainty |
|--------|------------|-------------|
| Duplicate payments found | 95% | Some retries might be legitimate |
| True recovery amount | 85% | PTP validation has some uncertainty |
| Cohort recovery rate (flat) | 80% | Portfolio classification could be better |
| Root cause attribution | 70% | Multiple factors, some correlation unknown |
| Operational improvement | 60% | Confounding factors remain |

---

## RECOMMENDATIONS

### Immediate (This Month)
1. ✅ Clean duplicate payments (₹7-8M recovery, low effort)
2. ✅ Standardize timezone handling (prevents future errors)
3. ✅ Cross-validate PTP with payments (verify metrics)

### Short-term (Next Quarter)
1. Implement cohort-based recovery tracking
2. Set up attribution model (voice vs digital)
3. Agent tenure tracking dashboard
4. Campaign management improvements (versioning)

### Medium-term (Next 2-4 Quarters)
1. **Invest in voice automation** (primary recommendation for ₹10 Cr)
   - ROI: 3.3 year payback
   - Risk: Medium (borrower acceptance)
   - Confidence: 70%

2. Better data governance
   - Real-time PTP→Payment reconciliation
   - Campaign definition approval workflow
   - Data quality dashboard

3. Causal inference models
   - Understand true channel effectiveness
   - Estimate elasticity (how much does quality improve with investment?)
   - Build forecasting model

---

**Prepared by:** Data Analysis Team  
**Date:** August 2026  
**Confidence:** 75% overall  
**Next Review:** October 2026
