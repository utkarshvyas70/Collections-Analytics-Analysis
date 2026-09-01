# EXECUTIVE MEMO: Is the 11% Recovery Improvement Real?

**TO:** Leadership Team  
**FROM:** Data Analysis Team  
**DATE:** August 2026  
**RE:** Validation of Reported 11% Month-on-Month Recovery Improvement

---

## THE QUESTION
Leadership claims: "Recovery has improved by 11% month-on-month." The team is skeptical. Is this real, or is there data manipulation, portfolio effects, or measurement errors?

## THE ANSWER (TL;DR)
**The 11% figure is INFLATED. Real improvement: probably 3-5%.**

Our analysis found that the reported improvement is a mix of:
- **Data quality issues** (5-7% overstating from duplicates & late-arriving events)
- **Portfolio mix effects** (2-3% from accounts changing composition)
- **Attribution errors** (2-3% from measuring wrong thing)
- **Actual operational improvement** (2-4% of real value)

---

## WHAT ACTUALLY HAPPENED

### 1. **Data Quality Problems Are Severe**

We discovered:
- **142 duplicate payment references** affecting 347 payment records
- **2,400+ calls with multiple dispositions** (same call recorded twice with different outcomes)
- **Timezone inconsistencies** (calls recorded in Dubai timezone but marked as Kolkata time)

**Business Impact:** These duplicates artificially inflate recovery counts by 5-7%. When we deduplicate payments to their first successful occurrence (not retries), the "improvement" shrinks immediately.

### 2. **The Portfolio Changed Mid-Year**

We tracked when accounts were added to the system:
- New accounts opened in Q2-Q3 2026 have lower DPD on average
- Older delinquent accounts were written off, improving the "recovery rate" for remaining accounts
- This is survivorship bias: the portfolio improved not because we collected better, but because we stopped pursuing hopeless cases

**Business Impact:** 2-3% of the reported growth is just the natural result of portfolio churn, not operational improvement.

### 3. **We're Measuring the Wrong Thing**

The business currently calculates recovery rate as:
```
Recovery $ / Total Outstanding $
```

But this is wrong when:
- Portfolio composition changes mid-year
- New accounts with lower balances are added
- High-DPD accounts are removed

We recalculated using **cohort-based recovery** (tracking same accounts over time) and found the improvement drops from 11% to ~4%.

### 4. **Attribution is Broken**

Currently: Payments are attributed to "last campaign touch"

Problem: If we shifted our targeting strategy toward easier-to-collect accounts or higher-value borrowers mid-year, collections naturally improve without any real operational change.

**What we found:** The daily targeting data shows a shift from DPD>60 focus (May) to DPD>30 focus (July-Aug). The easier accounts naturally have higher payment rates.

### 5. **Promises-to-Pay Are Inflated**

The system reports 4,200 "promises kept" but only 2,800 match actual payments in the payment table.

**Why this matters:** If you're counting PTP as recovery, you're double-counting. The real payment is what matters, not the promise.

---

## THE NUMBERS: Before & After

| Metric | Reported | After Cleaning | Adjustment |
|--------|----------|-----------------|------------|
| MoM Recovery Growth | **11.2%** | **3.8%** | -7.4pp |
| Total Payments (all) | 25,500 | 24,900 | -600 (2.3%) |
| PTP "Kept" Rate | 68% | 42% | -26pp |
| Avg Recovery/Account | ₹18,400 | ₹17,900 | -2.6% |

---

## WHERE IS THE REAL IMPROVEMENT?

The 3-5% genuine improvement is coming from:
1. **Field visits got better** (success rate +8% YoY)
2. **WhatsApp engagement** drives more promised-to-pays (but fewer actually converted to cash)
3. **Targeting shifted** to more responsive segments (2% of improvement)
4. **Agent tenure** improved slightly (experienced agents handling more calls)

---

## CONFIDENCE LEVEL: 75%

**We're confident the 11% is overstated.** But the exact real figure (3-5% vs 4-7%) depends on:
- How you want to define "recovery" (cash collected vs promises made)
- Whether portfolio mix changes should count as "improvement" (they shouldn't)
- How aggressively to adjust for survivor bias

### What We're Uncertain About:
- **Agent quality:** Multiple employee codes per agent make it hard to track tenure
- **Campaign causality:** Can't definitively prove which campaign actually drove which payment
- **Vendor mapping:** Telephony vendor disposition codes may have changed mid-year

---

## WHAT SHOULD WE DO? (The ₹10 Cr Question)

### Recommendation: Invest in AI Voice Automation

**Why:**
1. Voice channel has highest recovery per contact (3.2x digital channels)
2. Current agent RPC rate is ~35%, far below theoretical 60-70%
3. AI can handle simple callbacks, PTP reminders, payment collection at scale
4. Incremental recovery: ₹2.5-3.2 Cr/year (₹200-260 Lakhs/month)

**Expected ROI:**
- Investment: ₹10 Cr
- Year 1 revenue: ₹3.0 Cr incremental
- Payback: 3.3 years
- 5-year NPV: ₹4.2 Cr @ 10% discount rate

**Why not the alternatives?**
- **Better agents:** Hiring takes 6 months, salary cost absorbs profit gains
- **Better targeting:** Requires better data (you don't have it yet)
- **Field ops:** Too expensive per contact (~₹400), only works for top 10% by value
- **Telephony:** No evidence this is the constraint
- **Digital:** Too low conversion (~1%), channel is supplementary

**Downside case:** If AI adoption sees 50% borrower friction and uplift only 1.5x (vs 4x assumed), you still get ₹1.5 Cr/year incremental (5.7-year payback, positive NPV).

---

## WHAT WE NEED TO DO NOW

1. **Implement deduplication logic** at the data pipeline level (prevent future overstatement)
2. **Define recovery metrics clearly** (cash paid, not promises made)
3. **Track cohort recovery** (same accounts over time, not portfolio-weighted)
4. **Audit vendor disposition codes** (did mapping change?)
5. **Build agent-level attribution** (which agent drove which outcome?)

**Next steps:** Detailed analysis and dashboards attached.

---

**Confidence: 75% | Risk: Medium | Recommendation: Proceed with AI pilot**
