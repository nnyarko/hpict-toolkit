# Analytics Test Pack - Eligibility integrity and services not rendered

*paying for the wrong person, the wrong period, or a service that never happened*

*Generated 2026-11-11 from the HPI-CT control library (`data/controls.csv`, 173 controls). 16 detection tests, paired to 16 primary controls. Reference patterns, not programs.*

## Before you run anything

**These queries produce review candidates, never payment or denial decisions.** Every hit is a question for a human.

**The toolkit never receives your data.** These are patterns you run inside your own environment. No claim data, no PHI, no identifiers leave your systems.

**Every query is a pattern, not a program.** Field names are generic; yours will differ. Expect to rewrite the FROM and JOIN clauses and to remap service categories to your own taxonomy.

**Every threshold is a starting point, not a standard.** Where a threshold is arithmetic it is labelled as such. Everywhere else, measure your own distribution first.

Read `PREFACE_how_to_run_these.md` before the first run.

## Scope of this family

Two failure surfaces that share a data signature. **Eligibility integrity** - was this person
enrolled, alive, at liberty, and not simultaneously enrolled somewhere else on the date of
service, and was another payer primary? **Services not rendered** - did the encounter happen at
all? They belong together because both produce the same kind of evidence: a claim that cannot
be reconciled against an independent record of a person or an event. Eligibility is also where
the largest dollars sit in Medicaid - PERM's eligibility component measures it directly, and
capitation compounds every error by paying monthly whether or not a service occurs.

This family carries a mirror risk that no other family in the pack has, and it belongs at the
front of the reader's mind rather than in a footnote. Every test here can be run in reverse. A
control that stops payment for an ineligible person is the same control that, mis-tuned,
terminates coverage for an eligible one - and a false-positive disenrollment is a program-
integrity failure *and* a beneficiary harm, not a conservative error. The queries below are
written to surface improper payments; the threshold guidance repeatedly notes where the same
logic, applied to a live eligibility decision rather than a paid claim, would do damage. Gap
note G2 records that the control library currently has no control on that side at all.

## Tests at a glance

| # | Test | Primary control | Also touches |
|---|---|---|---|
| 1 | Payment after date of death - fee-for-service and capitation | **MD-08** | MD-06, MD-16 |
| 2 | Services and capitation during incarceration | **MD-22** | MD-06, CD-17 |
| 3 | Concurrent enrollment - duplicate capitation in the same coverage month | **MD-09** | MD-07, MD-16 |
| 4 | Interstate duplicate enrollment - claims in two states in one period | **MD-07** | MD-09, PI-21 |
| 5 | Eligibility verification completeness at determination and redetermination | **MD-06** | MD-12, MD-13 |
| 6 | Claims paid outside the beneficiary's coverage span | **MD-12** | MD-06, MD-16 |
| 7 | Phantom encounter - a service with no surrounding care footprint | **CD-09** | PI-06, CD-03 |
| 8 | Benefit-integrity workflow coverage - leads that never became cases | **PI-06** | PI-14, MD-18 |
| 9 | Recruiter clustering and beneficiary-as-participant patterns | **PI-21** | MD-06, PI-22 |
| 10 | Third-party liability - Medicaid paid where another payer was liable | **PI-17** | CD-17, MD-16 |
| 11 | Duplicate payment across benefit periods and settings | **CD-17** | MD-17, PI-17 |
| 12 | Capitation paid with no encounter data - the silent member | **MD-17** | MD-16, MD-11 |
| 13 | Eligibility and enrollment data quality - impossible or missing attributes | **MD-16** | MD-17, MD-12 |
| 14 | Rapid disenrollment and agent-broker enrollment churn | **MC-16** | PI-21, MD-09 |
| 15 | Payment continued after a credible fraud allegation | **MD-18** | PI-16, PI-14 |
| 16 | Services billed under a deceased or excluded provider identity | **PE-09** | PE-08, PI-20 |

> **Control-ID validation:** every control referenced above resolves to a real ID in the current control library. See the gap notes at the end for controls this family *needs* but the library does not yet have.

---

## 1. Payment after date of death - fee-for-service and capitation

| | |
|---|---|
| **Paired control** | **MD-08** - Deceased-enrollee detection via SSA Death Master File match |
| **Control type** | Detective |
| **Module** | Module 07 - Medicaid & CHIP Integrity |
| **Owner** | PI data analyst |
| **Frequency** | Monthly |
| **System touchpoint** | T-MSIS/eligibility + DMF feed |
| **Evidence artifact** | DMF-match exception disposition |
| **Control failure mode** | Capitation/FFS paid for deceased enrollee |
| **Regulatory anchor** | 42 CFR 455.436 [binding]; SSA Death Master File [binding]; OIG A-04-21-09005 (24 Nov 2023) - capitation after enrollee deaths; recs 24-A-04-018.01 and .02 open, .03 (Death Master File match) closed [case evidence] |
| **Secondary controls** | MD-06 - Enrollee eligibility verification via authoritative electronic data …; MD-16 - T-MSIS monthly submission + data-quality validation |

**1 · Data hypothesis.** Where the Death Master File match is absent, run late, or matched only against eligibility
rather than against payments, claims and - more importantly - monthly capitation continue after
death. Capitation is the larger exposure by an order of magnitude in a managed-care programme,
because it pays silently every month with no service to trigger a review.

**2 · What to check.** Test both payment streams against the recorded date of death: FFS claim lines with a service
date after death, and capitation payments for a coverage month beginning after death. Split
each by whether the payment was made before or after the death record loaded - the two have
different remediations.

**3 · Reference SQL.**

```sql
-- TEST 1 - FFS and capitation paid after date of death
-- Generalises family 1's test 5 beyond home health to all service types, and adds the
-- capitation leg. Where both families are in use, THIS test supersedes that one.
-- Params: :period_start, :period_end, :grace_days (0)

-- 1a: fee-for-service
SELECT
    'FFS' AS payment_stream,
    cl.claim_line_id AS payment_id, cl.beneficiary_id,
    cl.billing_npi, cl.hcpcs, cl.service_date AS coverage_date,
    b.date_of_death, (cl.service_date - b.date_of_death) AS days_after_death,
    cl.paid_amount,
    CASE WHEN cl.payment_date > b.death_record_load_date
         THEN 'PAID_AFTER_DEATH_KNOWN' ELSE 'PAID_BEFORE_DEATH_KNOWN' END AS timing_class
FROM claim_lines cl
JOIN beneficiary b ON b.beneficiary_id = cl.beneficiary_id
WHERE b.date_of_death IS NOT NULL
  AND cl.service_date > b.date_of_death + :grace_days
  AND cl.service_date BETWEEN :period_start AND :period_end
  AND cl.adjudication_status = 'PAID'

UNION ALL

-- 1b: capitation - usually the larger exposure and the one nobody looks at
SELECT
    'CAPITATION' AS payment_stream,
    cap.capitation_id AS payment_id, cap.beneficiary_id,
    cap.plan_npi AS billing_npi, CAST(NULL AS VARCHAR) AS hcpcs,
    cap.coverage_month_start AS coverage_date,
    b.date_of_death, (cap.coverage_month_start - b.date_of_death) AS days_after_death,
    cap.paid_amount,
    CASE WHEN cap.payment_date > b.death_record_load_date
         THEN 'PAID_AFTER_DEATH_KNOWN' ELSE 'PAID_BEFORE_DEATH_KNOWN' END AS timing_class
FROM capitation_payment cap
JOIN beneficiary b ON b.beneficiary_id = cap.beneficiary_id
WHERE b.date_of_death IS NOT NULL
  AND cap.coverage_month_start > b.date_of_death
  AND cap.coverage_month_start BETWEEN :period_start AND :period_end
ORDER BY days_after_death DESC, paid_amount DESC;
```

**4 · Threshold guidance.** :grace_days = 0 for most services - but not all. A claim dated the day of death is usually
legitimate (the patient was treated, then died), and some facility and hospice claims
legitimately span the date of death, so exclude the death date itself and handle span-dated
claims on their end date rather than their start. The `timing_class` split is what makes this
actionable: PAID_BEFORE_DEATH_KNOWN is a recovery workflow question, PAID_AFTER_DEATH_KNOWN is
a control failure - the payer already had the information and paid anyway. Report the two
separately or the control finding is diluted by unavoidable data latency. Tune to your
population; a starting point, not a standard.

**5 · Paired control + failure mode.** MD-08 (Deceased-enrollee detection via SSA Death Master File match). Secondary: MD-06 (Enrollee eligibility verification), MD-16 (T-MSIS monthly submission + data-quality validation).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No DMF match runs. MD-08's DMF-match exception disposition does not exist. |
| **Designed-not-operating** | MD-08 is monthly; the match did not run for one or more months. Establish what was paid in the gap. |
| **Operated-but-bypassed** | Matches were dispositioned as false positives in bulk without a documented identity-resolution step. |
| **Operated-but-ineffective** | The match updates eligibility but never sweeps paid claims or capitation, so it prevents future payment without recovering past payment - the defining defect. |
| **Evidence-missing** | The match runs but no dated exception output or per-case disposition is retained. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 2. Services and capitation during incarceration

| | |
|---|---|
| **Paired control** | **MD-22** - Incarceration/institutional-status match - suspend claims/capitation during incarceration |
| **Control type** | Detective |
| **Module** | Module 07 - Medicaid & CHIP Integrity |
| **Owner** | PI data analyst / eligibility unit |
| **Frequency** | Monthly |
| **System touchpoint** | Eligibility system + corrections data feed |
| **Evidence artifact** | Monthly incarceration-match exception report + dispositions |
| **Control failure mode** | PCA/home-care billing while recipient incarcerated (pattern from a 2026 state MFCU case); capitation for incarcerated enrollees |
| **Regulatory anchor** | SSA §1905(a), clause (A) of the text after paragraph (32) [binding]; 42 CFR 435.1009–.1010 [binding]; CMS SHO 16-007/24-004 [guidance]; CAA 2024 (suspend-not-terminate, eff. 1/1/2026) [binding] |
| **Secondary controls** | MD-06 - Enrollee eligibility verification via authoritative electronic data …; CD-17 - Coordination-of-benefits / setting documentation control to prevent … |

**1 · Data hypothesis.** Federal financial participation is unavailable for most services to an inmate of a public
institution. Where the corrections-roster match is absent or runs only against eligibility,
both FFS claims and capitation continue through a custody period. Note the 2026 change: the CAA
2024 requires states to *suspend* rather than terminate eligibility, which makes the payment-
side control more important, not less - the person stays enrolled, so only a payment control
stops the money.

**2 · What to check.** Interval-join FFS claims and capitation months against the corrections roster, excluding the
inpatient-hospitalisation exception where FFP remains available, and excluding boundary days.

**3 · Reference SQL.**

```sql
-- TEST 2 - Claims and capitation inside a custody window
-- Generalises family 1's test 6 beyond home health and adds capitation.
-- SSA §1905(a)(29)(A); 42 CFR 435.1009-.1010; CAA 2024 suspend-not-terminate (eff. 1/1/2026).
-- Params: :period_start, :period_end

SELECT
    'FFS' AS payment_stream,
    cl.claim_line_id AS payment_id, cl.beneficiary_id, cl.billing_npi,
    cl.hcpcs, cl.service_date AS coverage_date, cl.paid_amount,
    inc.book_date, inc.release_date, inc.facility_id
FROM claim_lines cl
JOIN incarceration_roster inc
  ON  inc.beneficiary_id = cl.beneficiary_id
  AND cl.service_date >  inc.book_date                       -- exclude the booking day
  AND cl.service_date <  COALESCE(inc.release_date, DATE '9999-12-31')
WHERE cl.service_date BETWEEN :period_start AND :period_end
  AND cl.adjudication_status = 'PAID'
  -- the inpatient exception: FFP remains available for an inmate admitted as an
  -- inpatient to a medical institution for >24 hours. Exclude those stays.
  AND NOT EXISTS (
    SELECT 1 FROM institutional_stay st
    WHERE st.beneficiary_id = cl.beneficiary_id
      AND st.facility_type  = 'INPATIENT_HOSPITAL'
      AND cl.service_date BETWEEN st.admit_date AND COALESCE(st.discharge_date, DATE '9999-12-31')
      AND COALESCE(st.discharge_date, CURRENT_DATE) - st.admit_date >= 1
  )

UNION ALL

SELECT
    'CAPITATION' AS payment_stream,
    cap.capitation_id, cap.beneficiary_id, cap.plan_npi,
    CAST(NULL AS VARCHAR), cap.coverage_month_start, cap.paid_amount,
    inc.book_date, inc.release_date, inc.facility_id
FROM capitation_payment cap
JOIN incarceration_roster inc
  ON  inc.beneficiary_id = cap.beneficiary_id
  -- capitation month fully inside custody: a partial month is not a finding
  AND cap.coverage_month_start >= inc.book_date
  AND cap.coverage_month_end   <= COALESCE(inc.release_date, DATE '9999-12-31')
WHERE cap.coverage_month_start BETWEEN :period_start AND :period_end
ORDER BY paid_amount DESC;
```

**4 · Threshold guidance.** Categorical, with three carve-outs that must be built in before anyone acts: the inpatient-
hospitalisation exception (FFP continues for an inmate admitted as a hospital inpatient), the
boundary days around booking and release, and partial capitation months - a person booked on
the 20th was legitimately covered for nineteen days. Without these the output is dominated by
correct payments. Corrections-roster data quality is the other constraint: matching is usually
on name and date of birth rather than a shared identifier, so treat matches as candidates
requiring identity resolution, and record the match rule you used. Tune to your population; a
starting point, not a standard.

**5 · Paired control + failure mode.** MD-22 (Incarceration/institutional-status match - suspend claims/capitation during incarceration). Secondary: MD-06 (Enrollee eligibility verification), CD-17 (Coordination-of-benefits / setting documentation control).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No corrections-data feed exists. MD-22's monthly exception report does not exist. |
| **Designed-not-operating** | MD-22 is monthly; the roster match did not run, or ran without the paid-claims join its design specifies. |
| **Operated-but-bypassed** | Conflicts were closed on an unverified assertion of early release. |
| **Operated-but-ineffective** | The match suspends future capitation only and never tests already-paid claims. Under suspend-not-terminate this is a larger defect than it was before 2026. |
| **Evidence-missing** | Match runs but no dated exception report with dispositions is retained. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 3. Concurrent enrollment - duplicate capitation in the same coverage month

| | |
|---|---|
| **Paired control** | **MD-09** - Concurrent multi-state MCO enrollment detection |
| **Control type** | Detective |
| **Module** | Module 07 - Medicaid & CHIP Integrity |
| **Owner** | PI data analyst |
| **Frequency** | Monthly |
| **System touchpoint** | Enrollment analytics |
| **Evidence artifact** | Concurrent-enrollment alert log |
| **Control failure mode** | Duplicate capitation across states |
| **Regulatory anchor** | 42 CFR 438.608 [binding]; OIG A-05-20-00025 (19 Sep 2022) - concurrent capitation; both recommendations closed [case evidence] |
| **Secondary controls** | MD-07 - Interstate duplicate-enrollment matching (PARIS) - resolve & terminate; MD-16 - T-MSIS monthly submission + data-quality validation |

**1 · Data hypothesis.** A beneficiary should generate one capitation payment per coverage month. Where concurrent-
enrollment detection is absent, an identity enrolled in two managed care plans - or in two
states, or in both managed care and fee-for-service - generates duplicate payment every month
until someone notices. Nothing in the claim stream signals it, because no claim is wrong.

**2 · What to check.** Net each reversal and adjustment against the payment it belongs to, then group the payments
still standing by beneficiary and coverage month and flag any month with more than one,
reporting whether the duplication is across plans, across states, or across delivery systems.
Add the FFS-alongside-capitation case, which is the same failure in a different shape.

**3 · Reference SQL.**

```sql
-- TEST 3 - More than one capitation payment for one beneficiary-month
-- Params: :period_start, :period_end

-- Reversals and adjustments are netted against the payment they belong to BEFORE counting,
-- in two steps:
--   1. a row that carries adjusts_capitation_id (a reversal or a retroactive rate adjustment
--      that names its payment) is added to that payment and is not a payment of its own;
--   2. a negative row with no reference cancels one payment of the same amount for the same
--      beneficiary, month and plan.
-- What is left is the set of payments still standing for the month.
WITH cap AS (
  SELECT c.capitation_id, c.beneficiary_id, c.coverage_month_start, c.plan_npi, c.state_code,
         c.paid_amount, c.adjusts_capitation_id
  FROM capitation_payment c
  WHERE c.coverage_month_start BETWEEN :period_start AND :period_end
),
referenced AS (
  SELECT adjusts_capitation_id AS capitation_id, SUM(paid_amount) AS adjustment_amount
  FROM cap
  WHERE adjusts_capitation_id IS NOT NULL
  GROUP BY adjusts_capitation_id
),
netted AS (
  SELECT c.capitation_id, c.beneficiary_id, c.coverage_month_start, c.plan_npi, c.state_code,
         ROUND(c.paid_amount + COALESCE(r.adjustment_amount, 0), 2) AS net_amount
  FROM cap c
  LEFT JOIN referenced r ON r.capitation_id = c.capitation_id
  WHERE c.adjusts_capitation_id IS NULL
),
numbered AS (                -- number equal amounts so one reversal cancels exactly one payment
  SELECT n.*,
         ROW_NUMBER() OVER (PARTITION BY n.beneficiary_id, n.coverage_month_start, n.plan_npi,
                                         ABS(n.net_amount),
                                         CASE WHEN n.net_amount > 0 THEN 1 ELSE 0 END
                            ORDER BY n.capitation_id) AS seq
  FROM netted n
  WHERE n.net_amount <> 0
),
standing AS (
  SELECT p.*
  FROM numbered p
  WHERE p.net_amount > 0
    AND NOT EXISTS (SELECT 1 FROM numbered v
                    WHERE v.net_amount < 0
                      AND v.beneficiary_id       = p.beneficiary_id
                      AND v.coverage_month_start = p.coverage_month_start
                      AND v.plan_npi             = p.plan_npi
                      AND ABS(v.net_amount)      = p.net_amount
                      AND v.seq                  = p.seq)
),
bene_month AS (
  SELECT
      s.beneficiary_id,
      s.coverage_month_start,
      COUNT(*)                            AS payment_count,           -- payments still standing
      COUNT(DISTINCT s.plan_npi)          AS distinct_plans,
      COUNT(DISTINCT s.state_code)        AS distinct_states,
      SUM(s.net_amount)                   AS total_paid,              -- net of reversals and adjustments
      STRING_AGG(DISTINCT s.plan_npi, ', ') AS plans                  -- LISTAGG on Oracle and Snowflake
  FROM standing s
  GROUP BY s.beneficiary_id, s.coverage_month_start
)
SELECT
    bm.*,
    CASE
      WHEN distinct_states > 1 THEN 'CROSS_STATE'
      WHEN distinct_plans  > 1 THEN 'CROSS_PLAN'
      ELSE 'SAME_PLAN_DUPLICATE'
    END AS duplication_class,
    -- FFS paid in the same month as capitation: the same failure in another shape
    (SELECT COALESCE(SUM(cl.paid_amount), 0)
       FROM claim_lines cl
      WHERE cl.beneficiary_id = bm.beneficiary_id
        AND cl.adjudication_status = 'PAID'
        AND cl.payment_arrangement = 'FFS'
        -- last day of the month: PostgreSQL and DuckDB interval syntax; elsewhere use
        -- DATEADD(day, -1, DATEADD(month, 1, month_start)) or LAST_DAY(month_start)
        AND cl.service_date BETWEEN bm.coverage_month_start
                                AND bm.coverage_month_start + INTERVAL '1 month' - INTERVAL '1 day'
    ) AS ffs_paid_same_month
FROM bene_month bm
WHERE payment_count > 1                  -- more than one payment still standing after netting
ORDER BY total_paid DESC;

-- Run a second pass for beneficiaries with capitation AND material FFS spend in the same
-- month even where capitation is singular - carve-out services are legitimate, so build
-- the carve-out code list first and exclude it, or this pass is unusable.
```

**4 · Threshold guidance.** Categorical for duplicate capitation, but the identity question sits underneath it: cross-state
and cross-plan duplication usually means one person holds two enrollment records, and
confirming that is identity resolution, not analytics. Retroactive enrollment adjustments and
mid-month plan changes both generate legitimate multiple payments in a month - check how your
capitation table records adjustments and reversals before treating a count above one as
duplication, because a payment and its reversal will otherwise read as two payments. The query
nets them in two ways: a row that names the payment it adjusts (adjusts_capitation_id) is added
to that payment, and an unreferenced negative row cancels one payment of the same amount in the
same plan and month. A partial reversal or a positive adjustment that carries no reference
cannot be matched and still counts, so populate the reference where your source has one. Tune
to your population; a starting point, not a standard.

**5 · Paired control + failure mode.** MD-09 (Concurrent multi-state MCO enrollment detection). Secondary: MD-07 (Interstate duplicate-enrollment matching via PARIS), MD-16 (T-MSIS submission + data-quality validation).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No concurrent-enrollment scan exists. MD-09's concurrent-enrollment alert log does not exist. |
| **Designed-not-operating** | MD-09 is monthly; no scan ran for the period. |
| **Operated-but-bypassed** | Alerts were raised and neither enrollment was terminated nor capitation recovered. |
| **Operated-but-ineffective** | The scan compares enrollment records within the state only, so cross-state duplication - the case MD-09 is named for - is structurally invisible. |
| **Evidence-missing** | Alerts generated with no retained disposition record. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 4. Interstate duplicate enrollment - claims in two states in one period

| | |
|---|---|
| **Paired control** | **MD-07** - Interstate duplicate-enrollment matching (PARIS) - resolve & terminate |
| **Control type** | Detective |
| **Module** | Module 07 - Medicaid & CHIP Integrity |
| **Owner** | PI / eligibility data unit |
| **Frequency** | Quarterly |
| **System touchpoint** | PARIS interface |
| **Evidence artifact** | Match-resolution & termination log |
| **Control failure mode** | Duplicate cross-state enrollment/benefits |
| **Regulatory anchor** | 42 CFR 435.945(d) (State eligibility determination systems must conduct PARIS data matching) [binding]; GAO-25-106976 (25 Sep 2025) Rec 3 to CMS, open: require State Medicaid/CHIP agencies to submit all enrollment data to PARIS frequently, review matches and end coverage when appropriate [case evidence, advisory to CMS] |
| **Secondary controls** | MD-09 - Concurrent multi-state MCO enrollment detection; PI-21 - Beneficiary-side integrity analytics - detect recruiter-clustered … |

**1 · Data hypothesis.** PARIS matching exists to find beneficiaries enrolled in more than one state. Where the
quarterly match is absent or matches are never resolved to a termination, the duplicate
persists. GAO has an open recommendation on exactly this. A payer with multi-state data can see
the pattern directly: service claims in two states within a window too short to represent a
genuine relocation.

**2 · What to check.** For each beneficiary identity, find claim or capitation activity in more than one state within
a rolling window, and rank by the overlap duration and the paid amount in the overlapping
period.

**3 · Reference SQL.**

```sql
-- TEST 4 - Activity in two states within one window (PARIS-style)
-- Requires either multi-state data or a matched identity across states. Where you hold
-- only one state's data, this test is not runnable - and MD-07's PARIS interface is the
-- control that exists precisely because of that limitation.
-- Params: :period_start, :period_end, :overlap_window_days (60)

WITH activity AS (
  SELECT beneficiary_identity_hash, state_code,
         MIN(service_date) AS first_activity,
         MAX(service_date) AS last_activity,
         COUNT(*)          AS events,
         SUM(paid_amount)  AS paid_amount
  FROM (
      SELECT cl.beneficiary_identity_hash, cl.state_code, cl.service_date, cl.paid_amount
      FROM claim_lines cl
      WHERE cl.service_date BETWEEN :period_start AND :period_end
        AND cl.adjudication_status = 'PAID'
      UNION ALL
      SELECT cap.beneficiary_identity_hash, cap.state_code,
             cap.coverage_month_start, cap.paid_amount
      FROM capitation_payment cap
      WHERE cap.coverage_month_start BETWEEN :period_start AND :period_end
  ) u
  GROUP BY beneficiary_identity_hash, state_code
)
SELECT
    a.beneficiary_identity_hash,
    a.state_code AS state_a, b.state_code AS state_b,
    a.first_activity AS a_first, a.last_activity AS a_last,
    b.first_activity AS b_first, b.last_activity AS b_last,
    a.events + b.events         AS combined_events,
    a.paid_amount + b.paid_amount AS combined_paid,
    -- days of genuine overlap between the two states' activity spans
    LEAST(a.last_activity, b.last_activity)
      - GREATEST(a.first_activity, b.first_activity) AS overlap_days
FROM activity a
JOIN activity b
  ON  b.beneficiary_identity_hash = a.beneficiary_identity_hash
  AND b.state_code > a.state_code                  -- each state pair once
  AND a.first_activity <= b.last_activity
  AND b.first_activity <= a.last_activity
WHERE LEAST(a.last_activity, b.last_activity)
      - GREATEST(a.first_activity, b.first_activity) > :overlap_window_days
ORDER BY combined_paid DESC;
```

**4 · Threshold guidance.** :overlap_window_days = 60 is deliberately generous. Relocation is normal, and a genuine move
produces exactly this pattern for a short transition period while the old state's coverage runs
out - so a short overlap is expected and the signal is duration. Border-region beneficiaries
who legitimately receive care across a state line will also appear; consider excluding
adjacent-state pairs in border counties, or at least stratifying by them. Identity matching
drives everything here, so record the hash rule you used and treat every hit as a candidate for
resolution. Tune to your population; a starting point, not a standard.

**5 · Paired control + failure mode.** MD-07 (Interstate duplicate-enrollment matching via PARIS - resolve & terminate). Secondary: MD-09 (Concurrent multi-state MCO enrollment detection), PI-21 (Beneficiary-side integrity analytics).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No PARIS participation or matching. MD-07's match-resolution and termination log does not exist. |
| **Designed-not-operating** | MD-07 is quarterly; one or more quarterly matches did not run. |
| **Operated-but-bypassed** | Matches were received and never worked to a resolution - the GAO open recommendation is specifically about resolution, not matching. |
| **Operated-but-ineffective** | Matches are received but the workflow has no path to terminate the duplicate enrollment, so identification never converts to action. |
| **Evidence-missing** | Matching occurs but no log records which matches were resolved and how. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 5. Eligibility verification completeness at determination and redetermination

| | |
|---|---|
| **Paired control** | **MD-06** - Enrollee eligibility verification via authoritative electronic data sources |
| **Control type** | Preventive |
| **Module** | Module 07 - Medicaid & CHIP Integrity |
| **Owner** | Eligibility operations |
| **Frequency** | Per determination |
| **System touchpoint** | Eligibility system / data hub |
| **Evidence artifact** | Verification record per data element |
| **Control failure mode** | Ineligible enrollee receives benefits |
| **Regulatory anchor** | 42 CFR Part 435 Subpart J (435.940-.965) - eligibility verification [binding] |
| **Secondary controls** | MD-12 - PERM / MEQC eligibility & claims sampling with error-rate measurement; MD-13 - Improper-payment corrective action plans addressing root causes, … |

**1 · Data hypothesis.** 42 CFR Part 435 Subpart J requires verification of specified eligibility data elements through
electronic data sources. Where the verification control is absent or partial, determinations
are made with elements unverified - and the resulting eligibility errors are the largest single
component of the Medicaid improper payment rate. The signature is in the eligibility system,
not the claim stream: determinations with no verification record per required element.

**2 · What to check.** For each determination and redetermination in the period, check for a verification record
against each required data element, and report completeness by element, by worker, and by
office. Then quantify the downstream payment exposure attached to incomplete determinations.

**3 · Reference SQL.**

```sql
-- TEST 5 - Verification completeness per required eligibility data element
-- 42 CFR 435.940-.965. The required element list is program- and state-specific;
-- drive it from a reference table rather than hardcoding.
-- Params: :period_start, :period_end

WITH determinations AS (
  SELECT d.determination_id, d.beneficiary_id, d.determination_date,
         d.determination_type,             -- 'INITIAL' | 'REDETERMINATION'
         d.worker_id, d.office_id, d.outcome
  FROM eligibility_determination d
  WHERE d.determination_date BETWEEN :period_start AND :period_end
    AND d.outcome = 'APPROVED'
),
required AS (
  -- the element list is not filtered here: each determination is tested against the elements
  -- in force on ITS OWN date (see the WHERE below), not on the reporting period
  SELECT re.element_name, re.verification_source, re.applies_to_type,
         re.effective_date, re.termination_date
  FROM required_verification_element re
)
SELECT
    d.determination_id, d.beneficiary_id, d.determination_date,
    d.determination_type, d.worker_id, d.office_id,
    r.element_name, r.verification_source,
    v.verification_id IS NOT NULL AS is_verified,
    v.verified_date,
    -- downstream exposure attached to this determination
    (SELECT COALESCE(SUM(cl.paid_amount), 0) FROM claim_lines cl
      WHERE cl.beneficiary_id = d.beneficiary_id
        AND cl.service_date >= d.determination_date
        AND cl.service_date <  d.determination_date + 365
        AND cl.adjudication_status = 'PAID') AS paid_12mo_after
FROM determinations d
CROSS JOIN required r
LEFT JOIN eligibility_verification v
       ON  v.determination_id = d.determination_id
       AND v.element_name     = r.element_name
WHERE (r.applies_to_type = 'ALL' OR r.applies_to_type = d.determination_type)
  AND r.effective_date <= d.determination_date
  AND COALESCE(r.termination_date, DATE '9999-12-31') >= d.determination_date
  AND v.verification_id IS NULL
ORDER BY paid_12mo_after DESC;

-- Roll up by element_name first, then by office_id, then by worker_id. An element
-- missing across every office is a SYSTEM configuration problem; one missing in a single
-- office is a training or workload problem. Fix the right thing.
```

**4 · Threshold guidance.** Completeness is categorical against the required-element list, and the required-element list is
where the judgement sits - build it from 42 CFR 435 Subpart J and your own state plan, not from
an assumption. Two cautions. First, reasonable-compatibility rules mean an element can be
properly resolved without a successful electronic match, so absence of a verification record is
not necessarily absence of verification - check how your system records the reasonable-
compatibility path. Second, and more important: this test's output must never be used to
disenroll anyone. It identifies a control weakness in how determinations were made, not a
conclusion that any individual is ineligible. Tune to your population; a starting point, not a
standard.

**5 · Paired control + failure mode.** MD-06 (Enrollee eligibility verification via authoritative electronic data sources). Secondary: MD-12 (PERM / MEQC eligibility & claims sampling), MD-13 (Improper-payment corrective action plans).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No electronic verification operates. MD-06's per-element verification record does not exist. |
| **Designed-not-operating** | MD-06 is per-determination; a population of approved determinations has no verification records. |
| **Operated-but-bypassed** | Verification failed or returned no match and the determination proceeded without the documented reasonable-compatibility or self-attestation path. |
| **Operated-but-ineffective** | Verification runs against a data source that is stale or does not cover the population, so it returns 'no match' routinely and workers learn to override it. |
| **Evidence-missing** | Verification occurred inside a vendor system with no record surfaced into the case file. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 6. Claims paid outside the beneficiary's coverage span

| | |
|---|---|
| **Paired control** | **MD-12** - PERM / MEQC eligibility & claims sampling with error-rate measurement |
| **Control type** | Detective |
| **Module** | Module 07 - Medicaid & CHIP Integrity |
| **Owner** | Program-integrity office |
| **Frequency** | Per cycle |
| **System touchpoint** | PERM/MEQC sampling system |
| **Evidence artifact** | Sampled-case workpapers + error rate |
| **Control failure mode** | Undetected improper-payment error rate |
| **Regulatory anchor** | 42 CFR Part 431 Subpart Q (PERM/MEQC) [binding]; CMS PERM program [guidance] |
| **Secondary controls** | MD-06 - Enrollee eligibility verification via authoritative electronic data …; MD-16 - T-MSIS monthly submission + data-quality validation |

**1 · Data hypothesis.** The simplest eligibility test there is, and the one most often assumed to be handled by
adjudication. Where eligibility spans are loaded late, retroactively adjusted, or held
inconsistently between the eligibility system and the claims system, claims pay for dates on
which the person was not enrolled. PERM measures this as an eligibility error; the payer can
measure it directly.

**2 · What to check.** Join every paid claim line to the beneficiary's eligibility spans and flag service dates
falling in no span, distinguishing claims that were outside the span at adjudication from those
made non-conforming by a later retroactive change.

**3 · Reference SQL.**

```sql
-- TEST 6 - Service dates with no covering eligibility span
-- Params: :period_start, :period_end

SELECT
    cl.claim_line_id, cl.beneficiary_id, cl.billing_npi,
    cl.hcpcs, cl.service_date, cl.paid_amount, cl.payment_date,
    -- nearest spans either side, to characterise the gap
    (SELECT MAX(e.span_end) FROM eligibility_span e
      WHERE e.beneficiary_id = cl.beneficiary_id AND e.span_end < cl.service_date)
      AS prior_span_end,
    (SELECT MIN(e.span_start) FROM eligibility_span e
      WHERE e.beneficiary_id = cl.beneficiary_id AND e.span_start > cl.service_date)
      AS next_span_start,
    CASE WHEN EXISTS (
      SELECT 1 FROM eligibility_span_history h
      WHERE h.beneficiary_id = cl.beneficiary_id
        AND h.change_date > cl.payment_date
        AND cl.service_date BETWEEN h.old_span_start AND COALESCE(h.old_span_end, DATE '9999-12-31')
    ) THEN 'RETROACTIVE_CHANGE' ELSE 'INELIGIBLE_AT_ADJUDICATION' END AS gap_class
FROM claim_lines cl
WHERE cl.service_date BETWEEN :period_start AND :period_end
  AND cl.adjudication_status = 'PAID'
  AND NOT EXISTS (
    SELECT 1 FROM eligibility_span e
    WHERE e.beneficiary_id = cl.beneficiary_id
      AND cl.service_date BETWEEN e.span_start
                              AND COALESCE(e.span_end, DATE '9999-12-31')
  )
ORDER BY gap_class, cl.paid_amount DESC;
```

**4 · Threshold guidance.** Categorical, and the `gap_class` split is the entire value of the test. RETROACTIVE_CHANGE
claims were correct when paid and became non-conforming afterward - that is a recovery and
reconciliation question, and in the case of a retroactive *termination* it may also be a
beneficiary-notice question worth asking about. INELIGIBLE_AT_ADJUDICATION is the control
finding: the eligibility data the claims system held was wrong or absent at the moment it
mattered. Merging the two makes both uninterpretable. Expect a large RETROACTIVE_CHANGE bucket
in any programme with retroactive eligibility; that is the system working as designed, not a
finding. Tune to your population; a starting point, not a standard.

**5 · Paired control + failure mode.** MD-12 (PERM / MEQC eligibility & claims sampling with error-rate measurement). Secondary: MD-06 (Enrollee eligibility verification), MD-16 (T-MSIS submission + data-quality validation).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No eligibility-error measurement operates outside the federal PERM cycle. MD-12's sampled-case workpapers do not exist for the period. |
| **Designed-not-operating** | The MEQC pilot or equivalent did not run in its cycle. |
| **Operated-but-bypassed** | Errors were measured and no corrective action plan followed (MD-13). |
| **Operated-but-ineffective** | Sampling covers claims accuracy only and never tests the eligibility determination behind the claim, so the eligibility error category is out of scope by construction. |
| **Evidence-missing** | Sampling occurred but workpapers do not evidence the eligibility basis tested. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 7. Phantom encounter - a service with no surrounding care footprint

| | |
|---|---|
| **Paired control** | **CD-09** - Proof-of-delivery / proof-of-service capture and retention with all required elements (DMEPOS) |
| **Control type** | Detective |
| **Module** | Module 05 - Coverage & Documentation |
| **Owner** | Supplier operations |
| **Frequency** | Per delivery |
| **System touchpoint** | POD system |
| **Evidence artifact** | Complete POD record |
| **Control failure mode** | Billing for items not delivered / POD incomplete (42 CFR 424.57(c)(12)) |
| **Regulatory anchor** | Program Integrity Manual (Pub 100-08) Ch.5 sec.5.10 and Ch.4 sec.4.7.3.1 [guidance]; 42 CFR 424.57(c)(12) (supplier must maintain proof of delivery) [binding] |
| **Secondary controls** | PI-06 - Medical review for benefit integrity (suspected-fraud) workflow; CD-03 - Reasonable-and-necessary documentation control: the medical record … |

**1 · Data hypothesis.** A real clinical encounter usually leaves a footprint: a referral before it, a prescription or
follow-up after it, other claims from the same provider for the same person. Where proof-of-
service capture is absent, billing for services never rendered produces claims that are
isolated - a single high-value encounter with a provider the beneficiary has no other
relationship with, and nothing before or after.

**2 · What to check.** For each paid encounter, count the beneficiary's other claims with the same provider in a
surrounding window and any downstream activity (pharmacy, referral, follow-up). Flag isolated
high-value encounters and rank providers by the share of their volume that is isolated.

**3 · Reference SQL.**

```sql
-- TEST 7 - Isolated encounters with no surrounding care footprint
-- A prioritisation signal, not evidence. Isolated encounters are legitimately common
-- (urgent care, one-off consults, screening). The PROVIDER-level rate is the signal.
-- Params: :period_start, :period_end, :footprint_window_days (90), :min_paid (200), :min_isolated (3)

WITH encounters AS (
  SELECT
      cl.claim_line_id, cl.beneficiary_id, cl.rendering_npi, cl.billing_npi,
      cl.hcpcs, cl.service_date, cl.paid_amount,
      (SELECT COUNT(*) FROM claim_lines o
        WHERE o.beneficiary_id = cl.beneficiary_id
          AND o.rendering_npi  = cl.rendering_npi
          AND o.claim_line_id <> cl.claim_line_id
          AND o.adjudication_status = 'PAID'
          AND o.service_date BETWEEN cl.service_date - :footprint_window_days
                                 AND cl.service_date + :footprint_window_days)
        AS same_provider_claims,
      (SELECT COUNT(*) FROM pharmacy_claim rx
        WHERE rx.beneficiary_id = cl.beneficiary_id
          AND rx.fill_date BETWEEN cl.service_date AND cl.service_date + :footprint_window_days)
        AS downstream_rx,
      (SELECT COUNT(*) FROM claim_lines f
        WHERE f.beneficiary_id = cl.beneficiary_id
          AND f.rendering_npi <> cl.rendering_npi
          AND f.adjudication_status = 'PAID'
          AND f.service_date BETWEEN cl.service_date AND cl.service_date + :footprint_window_days)
        AS downstream_other_provider
  FROM claim_lines cl
  WHERE cl.service_date BETWEEN :period_start AND :period_end
    AND cl.adjudication_status = 'PAID'
    AND cl.paid_amount >= :min_paid
),
isolated AS (
  SELECT * FROM encounters
  WHERE same_provider_claims = 0
    AND downstream_rx = 0
    AND downstream_other_provider = 0
)
SELECT
    e.rendering_npi, pr.specialty,
    COUNT(*)                                  AS isolated_encounters,
    SUM(e.paid_amount)                        AS isolated_paid,
    (SELECT COUNT(*) FROM encounters a WHERE a.rendering_npi = e.rendering_npi) AS total_encounters,
    ROUND(COUNT(*) * 1.0 /
          NULLIF((SELECT COUNT(*) FROM encounters a WHERE a.rendering_npi = e.rendering_npi), 0), 3)
      AS isolated_rate
FROM isolated e
LEFT JOIN provider pr ON pr.npi = e.rendering_npi
GROUP BY e.rendering_npi, pr.specialty
HAVING COUNT(*) >= :min_isolated
ORDER BY isolated_rate DESC, isolated_paid DESC;
```

**4 · Threshold guidance.** There is no defensible absolute threshold here - the isolated-encounter rate varies enormously
by specialty, and urgent care, screening, telehealth-only and mobile services are legitimately
isolated most of the time. Benchmark the rate within specialty and treat only the extreme tail
as a candidate. Two structural caveats: a beneficiary who receives care under another payer
will look isolated in your data, and a footprint window that is too short manufactures
isolation. This is the weakest test in the family on its own and one of the more useful ones as
an input to a composite. Tune to your population; a starting point, not a standard.

**5 · Paired control + failure mode.** CD-09 (Proof-of-delivery / proof-of-service capture and retention). Secondary: PI-06 (Medical review for benefit integrity), CD-03 (Reasonable-and-necessary documentation).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No proof-of-service capture operates for this service category. CD-09's complete POD record does not exist outside DMEPOS. |
| **Designed-not-operating** | The control is defined but no proof-of-service evidence was captured for a population of paid services. |
| **Operated-but-bypassed** | Proof-of-service was accepted in a form that does not independently evidence the encounter (e.g. the provider's own billing record). |
| **Operated-but-ineffective** | CD-09 is scoped to DMEPOS delivery only, so professional services have no equivalent control at all - see gap note G1. |
| **Evidence-missing** | Services were rendered and documented but no retained artifact independently evidences delivery. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 8. Benefit-integrity workflow coverage - leads that never became cases

| | |
|---|---|
| **Paired control** | **PI-06** - Medical review for benefit integrity (suspected-fraud) workflow |
| **Control type** | Detective |
| **Module** | Module 02 - Program Integrity |
| **Owner** | Benefit-integrity unit |
| **Frequency** | Per case |
| **System touchpoint** | BI case system |
| **Evidence artifact** | MR-for-BI case file |
| **Control failure mode** | Fraud not developed for action/referral |
| **Regulatory anchor** | CMS PIM Pub 100-08 Ch.4 (Benefit Integrity) [guidance]; PIM Sec.1.8 - Medical Review for Benefit Integrity [guidance] |
| **Secondary controls** | PI-14 - Suspected-fraud referral to OIG/MFCU/law enforcement; MD-18 - Credible-allegation-of-fraud payment suspension + MFCU referral |

**1 · Data hypothesis.** This test measures the pipeline, not the claims. Where the medical-review-for-benefit-integrity
workflow is absent or under-resourced, analytic leads and complaints accumulate without
becoming developed cases, and nothing converts to a referral. The pattern is a widening gap
between leads generated and cases opened, and an ageing backlog nobody reports.

**2 · What to check.** Reconcile the lead intake log against the benefit-integrity case system and the referral log:
how many leads were generated, how many became cases, how many cases reached a disposition, how
many converted to referrals, and how old the unworked backlog is.

**3 · Reference SQL.**

```sql
-- TEST 8 - Lead-to-case-to-referral conversion and backlog ageing
-- Params: :period_start, :period_end, :stale_lead_days (90)

WITH leads AS (
  SELECT l.lead_id, l.lead_source, l.created_date, l.subject_npi, l.exposure_estimate
  FROM bi_lead l
  WHERE l.created_date BETWEEN :period_start AND :period_end
),
linked AS (
  SELECT
      l.*,
      c.case_id, c.opened_date, c.disposition, c.closed_date,
      r.referral_id, r.referred_date, r.referred_to,
      -- one marker per case, so a case with several referrals is counted once where it must be
      ROW_NUMBER() OVER (PARTITION BY c.case_id ORDER BY r.referral_id) AS case_row
  FROM leads l
  LEFT JOIN bi_case c     ON c.lead_id = l.lead_id
  LEFT JOIN fraud_referral r ON r.case_id = c.case_id
)
SELECT
    lead_source,
    COUNT(DISTINCT lead_id)                                    AS leads,
    COUNT(DISTINCT case_id)                                    AS became_cases,
    COUNT(DISTINCT CASE WHEN disposition IS NOT NULL THEN case_id END) AS reached_disposition,
    COUNT(DISTINCT CASE WHEN referral_id IS NOT NULL THEN case_id END) AS became_referrals,   -- cases referred
    ROUND(COUNT(DISTINCT CASE WHEN case_id IS NOT NULL THEN lead_id END) * 1.0
          / NULLIF(COUNT(DISTINCT lead_id), 0), 3)             AS lead_to_case_rate,
    COUNT(DISTINCT CASE WHEN case_id IS NULL
                         AND CURRENT_DATE - created_date > :stale_lead_days THEN lead_id END)
      AS stale_unworked_leads,
    SUM(CASE WHEN case_id IS NULL THEN exposure_estimate ELSE 0 END)
      AS unworked_exposure,
    ROUND(AVG(CASE WHEN opened_date IS NOT NULL AND case_row = 1
                   THEN opened_date - created_date END), 1)    AS mean_days_lead_to_case
FROM linked
GROUP BY lead_source
ORDER BY unworked_exposure DESC;
```

**4 · Threshold guidance.** No threshold - this is a management report and should be read as one. A low lead-to-case rate
is not itself a failure: a well-tuned analytic generates many leads and most should close on
triage, and a unit that opened a case on every lead would be badly run. What matters is whether
triage decisions are *recorded*. A lead with no case and no documented triage outcome is
indistinguishable from a lead nobody looked at, and `stale_unworked_leads` with its attached
exposure is the number that belongs in front of governance. If you cannot produce this query
because leads are not logged, that inability is the PI-06 finding. Tune to your population; a
starting point, not a standard.

**5 · Paired control + failure mode.** PI-06 (Medical review for benefit integrity - suspected-fraud workflow). Secondary: PI-14 (Suspected-fraud referral to OIG/MFCU/law enforcement), MD-18 (Credible-allegation-of-fraud payment suspension + MFCU referral).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No benefit-integrity case workflow exists. PI-06's MR-for-BI case file does not exist. |
| **Designed-not-operating** | The workflow exists but no cases were opened in the period despite lead volume. |
| **Operated-but-bypassed** | Cases were opened and closed without development or disposition to clear a backlog. |
| **Operated-but-ineffective** | Cases are developed but never convert to referral or suspension, so the workflow terminates in itself - check PI-14 and MD-18 alongside. |
| **Evidence-missing** | Leads are triaged informally with no logged decision, making the conversion rate unmeasurable. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 9. Recruiter clustering and beneficiary-as-participant patterns

| | |
|---|---|
| **Paired control** | **PI-21** - Beneficiary-side integrity analytics - detect recruiter-clustered enrollment and staff-roster / member-roll identity overlap (employee/owner appearing as a recipient of their own entity's services) |
| **Control type** | Detective |
| **Module** | Module 02 - Program Integrity |
| **Owner** | SIU / PI data analyst |
| **Frequency** | Monthly |
| **System touchpoint** | Enrollment + claims + HR roster join |
| **Evidence artifact** | Recruiter-cluster + roster-overlap exception report + dispositions |
| **Control failure mode** | Beneficiary as paid participant - cash-bribe enrollment / employee-as-recipient (patterns from a 2026 adult day-care scheme and a 2026 non-emergency transport scheme, both alleged) |
| **Regulatory anchor** | 42 CFR 455 Subpart A (Medicaid fraud detection/investigation) [binding]; 42 USC 1320a-7b(b) (Anti-Kickback Statute - beneficiary inducement) [binding] |
| **Secondary controls** | MD-06 - Enrollee eligibility verification via authoritative electronic data …; PI-22 - Entity-network cash-loop detection - follow-the-money analytics … |

**1 · Data hypothesis.** Schemes that pay beneficiaries to participate leave enrollment and claim patterns that no
individual claim reveals: enrollments clustered around one address, applications submitted by
one assister, beneficiaries appearing on a provider's own staff roster. Where beneficiary-side
analytics are absent, none of this is visible because every control looks at providers.

**2 · What to check.** Cluster beneficiaries by shared address hash, shared contact details, common application
assister, and common first-service provider, and flag clusters materially above the population
norm. Intersect the beneficiary roll with provider staff rosters on hashed identity.

**3 · Reference SQL.**

```sql
-- TEST 9 - Enrollment clustering and roster overlap
-- Generalises family 1's test 11 beyond home care. Hashed identity comparison happens
-- inside your own environment; no identifiers leave the payer.
-- Params: :period_start, :period_end, :cluster_threshold (8)

-- 9a: enrollment clusters by address / assister / first-service provider
SELECT
    'ENROLLMENT_CLUSTER' AS signal,
    d.address_hash,
    d.assister_id,
    fp.first_service_npi,
    COUNT(DISTINCT d.beneficiary_id) AS beneficiaries,
    MIN(d.determination_date)        AS first_enrollment,
    MAX(d.determination_date)        AS last_enrollment,
    SUM(fp.paid_first_year)          AS paid_first_year
FROM eligibility_determination d
LEFT JOIN LATERAL (
    SELECT cl.rendering_npi AS first_service_npi,
           SUM(cl.paid_amount) AS paid_first_year
    FROM claim_lines cl
    WHERE cl.beneficiary_id = d.beneficiary_id
      AND cl.service_date BETWEEN d.determination_date AND d.determination_date + 365
      AND cl.adjudication_status = 'PAID'
    GROUP BY cl.rendering_npi
    ORDER BY MIN(cl.service_date)
    LIMIT 1
) fp ON TRUE
WHERE d.determination_date BETWEEN :period_start AND :period_end
  AND d.outcome = 'APPROVED'
  -- exclude known congregate settings before running, or they dominate the output
  AND d.address_hash NOT IN (SELECT address_hash FROM congregate_address_exclusion)
GROUP BY d.address_hash, d.assister_id, fp.first_service_npi
HAVING COUNT(DISTINCT d.beneficiary_id) >= :cluster_threshold

UNION ALL

-- 9b: individuals on both the beneficiary roll and a provider's staff roster
SELECT
    'ROSTER_OVERLAP' AS signal,
    CAST(NULL AS VARCHAR), CAST(NULL AS VARCHAR),
    s.employer_billing_npi AS first_service_npi,
    COUNT(DISTINCT b.beneficiary_id) AS beneficiaries,
    CAST(NULL AS DATE), CAST(NULL AS DATE),
    SUM(cl.paid_amount) AS paid_first_year
FROM staff_roster s
JOIN beneficiary b ON b.identity_hash = s.identity_hash
LEFT JOIN claim_lines cl
       ON cl.beneficiary_id = b.beneficiary_id
      AND cl.billing_npi    = s.employer_billing_npi
      AND cl.adjudication_status = 'PAID'
GROUP BY s.employer_billing_npi
ORDER BY paid_first_year DESC;
```

**4 · Threshold guidance.** :cluster_threshold = 8 distinct beneficiaries sharing an address hash is a starting cut and is
useless without the congregate-address exclusion - group homes, shelters, assisted living and
large apartment buildings will otherwise be the entire output. Build that exclusion list first.
Shared assisters are expected: navigators and certified application counsellors exist to submit
applications in volume, so assister clustering is only interesting when it coincides with a
single first-service provider. Roster overlap needs identity resolution before any conclusion  - 
an employee can lawfully be a beneficiary. Tune to your population; a starting point, not a
standard.

**5 · Paired control + failure mode.** PI-21 (Beneficiary-side integrity analytics - recruiter-clustered enrollment and roster/member-roll overlap). Secondary: MD-06 (Enrollee eligibility verification), PI-22 (Entity-network cash-loop detection).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No enrollment+claims+roster join exists. PI-21's exception report does not exist. |
| **Designed-not-operating** | PI-21 is monthly; no report was produced for the period. |
| **Operated-but-bypassed** | Clusters were identified and dismissed as coincidence with no identity-resolution step recorded. |
| **Operated-but-ineffective** | The analytic runs on claims only and never joins enrollment or roster data, so the defining signals are unavailable to it. |
| **Evidence-missing** | Analysis performed with no retained exception report or dispositions. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 10. Third-party liability - Medicaid paid where another payer was liable

| | |
|---|---|
| **Paired control** | **PI-17** - Medicaid third-party-liability cost-avoidance & recovery |
| **Control type** | Preventive |
| **Module** | Module 02 - Program Integrity |
| **Owner** | TPL unit |
| **Frequency** | Continuous |
| **System touchpoint** | TPL / eligibility match |
| **Evidence artifact** | TPL cost-avoidance report |
| **Control failure mode** | Medicaid pays where another payer liable |
| **Regulatory anchor** | 42 CFR 433.135-139 (Medicaid TPL) [binding]; OIG A-05-21-00013 (20 Oct 2023) - Medicaid third-party liability; recs 24-A-05-006.03 and .05 to .08 open, .01, .02 and .04 closed [case evidence] |
| **Secondary controls** | CD-17 - Coordination-of-benefits / setting documentation control to prevent …; MD-16 - T-MSIS monthly submission + data-quality validation |

**1 · Data hypothesis.** Medicaid is the payer of last resort. Where third-party-liability cost-avoidance is absent or
the TPL resource file is stale, Medicaid pays claims for beneficiaries with known other
coverage, and the money is recovered slowly or not at all. OIG has an open recommendation here,
and the exposure is chronic rather than dramatic - which is precisely why it persists.

**2 · What to check.** Join paid claims to the TPL resource file on the service date, flag claims paid where an active
third-party resource existed, and report cost-avoidance opportunity by resource type and by
whether any recovery was subsequently pursued.

**3 · Reference SQL.**

```sql
-- TEST 10 - Claims paid despite an active third-party resource
-- 42 CFR 433.135-139. Params: :period_start, :period_end

SELECT
    cl.claim_line_id, cl.beneficiary_id, cl.billing_npi,
    cl.hcpcs, cl.service_date, cl.paid_amount,
    tpl.resource_type,                 -- 'COMMERCIAL' | 'MEDICARE' | 'CASUALTY' | 'OTHER'
    tpl.carrier_name, tpl.policy_effective, tpl.policy_termination,
    tpl.verified_date AS tpl_verified_date,
    CASE WHEN rec.recovery_id IS NOT NULL THEN 'RECOVERY_PURSUED'
         ELSE 'NO_RECOVERY_ACTIVITY' END AS recovery_status,
    CASE WHEN cl.cob_indicator IS NULL OR cl.cob_indicator = 'N'
         THEN 'NO_COB_APPLIED' ELSE 'COB_APPLIED' END AS cob_status
FROM claim_lines cl
JOIN tpl_resource tpl
  ON  tpl.beneficiary_id = cl.beneficiary_id
  AND cl.service_date BETWEEN tpl.policy_effective
                          AND COALESCE(tpl.policy_termination, DATE '9999-12-31')
LEFT JOIN tpl_recovery rec
       ON rec.claim_line_id = cl.claim_line_id
WHERE cl.service_date BETWEEN :period_start AND :period_end
  AND cl.adjudication_status = 'PAID'
  AND (cl.cob_indicator IS NULL OR cl.cob_indicator = 'N')
  -- exclude service categories where cost-avoidance is not required and pay-and-chase applies
  AND cl.service_category NOT IN (SELECT service_category FROM tpl_payandchase_exception)
ORDER BY cl.paid_amount DESC;

-- Roll up by resource_type and by tpl_verified_date age. A large exposure attached to
-- resources verified years ago is a TPL-file-currency finding; one attached to recently
-- verified resources is an adjudication-edit finding. Different fixes.
```

**4 · Threshold guidance.** Categorical against the resource file, and the file's currency is the whole question. A TPL
record that is years old may describe coverage that has lapsed, in which case Medicaid paid
correctly and the finding is spurious - so read the output stratified by `tpl_verified_date`
age before concluding anything. Federal rules require pay-and-chase rather than cost-avoidance
for certain categories (prenatal, preventive paediatric, and services subject to child-support
enforcement among them); build that exception list first or the output misattributes correct
behaviour as failure. Tune to your population; a starting point, not a standard.

**5 · Paired control + failure mode.** PI-17 (Medicaid third-party-liability cost-avoidance & recovery). Secondary: CD-17 (Coordination-of-benefits / setting documentation control), MD-16 (T-MSIS submission + data-quality validation).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No TPL cost-avoidance operates. PI-17's TPL cost-avoidance report does not exist. |
| **Designed-not-operating** | PI-17 is continuous; no cost-avoidance activity is evidenced for the period. |
| **Operated-but-bypassed** | TPL was identified at adjudication and the claim paid anyway, with no recovery pursued afterward. |
| **Operated-but-ineffective** | The TPL resource file is not refreshed, so the edit runs against coverage information too stale to act on and staff learn to ignore it. |
| **Evidence-missing** | Cost-avoidance occurs but is not reported in a form that quantifies it. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 11. Duplicate payment across benefit periods and settings

| | |
|---|---|
| **Paired control** | **CD-17** - Coordination-of-benefits / setting documentation control to prevent duplicate or incorrect payment across benefit periods |
| **Control type** | Detective |
| **Module** | Module 05 - Coverage & Documentation |
| **Owner** | Billing / PI |
| **Frequency** | Per claim |
| **System touchpoint** | Eligibility / COB check |
| **Evidence artifact** | COB documentation |
| **Control failure mode** | Duplicate/incorrect payment across settings (OIG hospice outpatient) |
| **Regulatory anchor** | OIG A-09-23-03024 (12 Nov 2024) - Part B hospital outpatient services that duplicate the hospice per diem [case evidence]; all six recommendations closed [case evidence] |
| **Secondary controls** | MD-17 - Managed-care encounter/payment data completeness monitoring + state …; PI-17 - Medicaid third-party-liability cost-avoidance & recovery |

**1 · Data hypothesis.** Some benefits are all-inclusive for a period - hospice, inpatient stays, certain waiver
programmes - and separately billed services during that period duplicate payment already made.
Where the coordination/setting control is absent, the duplication is invisible because the two
claims come from different providers in different systems and neither is individually wrong.
The OIG hospice-outpatient finding cited by CD-17 is the reference case.

**2 · What to check.** For each all-inclusive benefit period, look for separately billed services falling inside it
that should have been covered by the period payment, excluding the categories that are
legitimately carved out.

**3 · Reference SQL.**

```sql
-- TEST 11 - Separately billed services inside an all-inclusive benefit period
-- Params: :period_start, :period_end

SELECT
    bp.benefit_period_id, bp.benefit_type,       -- 'HOSPICE' | 'INPATIENT' | 'WAIVER_BUNDLE'
    bp.beneficiary_id, bp.period_start, bp.period_end,
    bp.period_payer_npi,
    cl.claim_line_id, cl.billing_npi AS separate_biller,
    cl.hcpcs, cl.service_date, cl.paid_amount,
    (SELECT SUM(p.paid_amount) FROM benefit_period_payment p
      WHERE p.benefit_period_id = bp.benefit_period_id) AS period_paid,
    CASE WHEN cl.billing_npi = bp.period_payer_npi
         THEN 'SAME_ENTITY' ELSE 'DIFFERENT_ENTITY' END AS biller_relationship
FROM benefit_period bp
JOIN claim_lines cl
  ON  cl.beneficiary_id = bp.beneficiary_id
  AND cl.service_date >  bp.period_start          -- exclude boundary days
  AND cl.service_date <  COALESCE(bp.period_end, DATE '9999-12-31')
  AND cl.adjudication_status = 'PAID'
WHERE bp.period_start <= :period_end                                   -- overlap, so a long stay that began
  AND COALESCE(bp.period_end, DATE '9999-12-31') >= :period_start      -- before the period is still examined
  AND cl.service_date BETWEEN :period_start AND :period_end
  -- carve-outs that are legitimately separately payable during the period
  AND cl.hcpcs NOT IN (SELECT hcpcs FROM benefit_period_carveout
                        WHERE benefit_type = bp.benefit_type)
  -- services unrelated to the terminal condition are separately payable under hospice
  AND NOT EXISTS (
    SELECT 1 FROM claim_diagnosis d
    JOIN unrelated_condition_dx u ON u.icd10_code = d.icd10_code
    WHERE d.claim_id = cl.claim_id AND u.benefit_type = bp.benefit_type
  )
ORDER BY cl.paid_amount DESC;
```

**4 · Threshold guidance.** Categorical once the carve-out lists are right, and they carry the entire test. Under hospice,
services unrelated to the terminal condition remain separately payable - determining
relatedness is a clinical judgement that a diagnosis-code exclusion list only approximates, so
treat DIFFERENT_ENTITY hits as a chart-review queue rather than an overpayment list.
SAME_ENTITY hits are the stronger candidates: a provider billing separately for a service
inside a period they are already being paid for has fewer available explanations. Build the
carve-out and unrelated-condition lists with clinical input before the first run. Tune to your
population; a starting point, not a standard.

**5 · Paired control + failure mode.** CD-17 (Coordination-of-benefits / setting documentation control). Secondary: MD-17 (Managed-care encounter/payment data completeness monitoring), PI-17 (Medicaid third-party-liability cost-avoidance & recovery).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No cross-benefit-period checking exists. CD-17's COB documentation artifact does not exist. |
| **Designed-not-operating** | CD-17 is per-claim; no checking is evidenced for the period. |
| **Operated-but-bypassed** | Conflicts were identified at adjudication and paid on override. |
| **Operated-but-ineffective** | The check tests overlapping benefit periods but has no carve-out or relatedness logic, so it either fires on everything and is switched off, or is scoped so narrowly it fires on nothing. |
| **Evidence-missing** | No retained record of conflict identification and disposition. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 12. Capitation paid with no encounter data - the silent member

| | |
|---|---|
| **Paired control** | **MD-17** - Managed-care encounter/payment data completeness monitoring + state corrective action |
| **Control type** | Detective |
| **Module** | Module 07 - Medicaid & CHIP Integrity |
| **Owner** | Managed care data unit |
| **Frequency** | Monthly |
| **System touchpoint** | Encounter data warehouse |
| **Evidence artifact** | Encounter-completeness reconciliation |
| **Control failure mode** | Incomplete encounter data hides FWA |
| **Regulatory anchor** | CMS T-MSIS [guidance]; 42 CFR 438.242(c)-(d) and 438.602(e) (encounter data submission; State validation; independent audit at least every 3 years) [binding]; OIG OEI-02-19-00180 (26 Mar 2021) - State-reported managed care payment data in T-MSIS incomplete and inaccurate, all three recommendations to CMS closed [case evidence] |
| **Secondary controls** | MD-16 - T-MSIS monthly submission + data-quality validation; MD-11 - State-directed payment & capitation actuarial-soundness oversight |

**1 · Data hypothesis.** Capitation buys access, so a member with no utilisation is not itself a problem. But where
encounter-data completeness monitoring is absent, a *plan* with an abnormally high share of
zero-encounter members is either not submitting encounter data, not delivering care, or
carrying members who do not exist. All three matter and the data cannot immediately distinguish
them - which is the point of monitoring the rate rather than the member.

**2 · What to check.** For each plan, compute the share of member-months with zero encounters, benchmark against other
plans in the same programme and region, and report the capitation paid against zero-encounter
member-months. Add a full-year view: members with twelve months of capitation and no encounter
at all.

**3 · Reference SQL.**

```sql
-- TEST 12 - Zero-encounter member-months and zero-utilisation members by plan
-- Params: :period_start, :period_end, :min_member_months (1000), :z_threshold (2.5)

WITH member_months AS (
  SELECT
      cap.plan_npi, cap.state_code, cap.program_type,
      cap.beneficiary_id, cap.coverage_month_start,
      cap.paid_amount,
      (SELECT COUNT(*) FROM encounter e
        WHERE e.beneficiary_id = cap.beneficiary_id
          AND e.plan_npi       = cap.plan_npi
          AND e.service_date BETWEEN cap.coverage_month_start AND cap.coverage_month_end)
        AS encounters
  FROM capitation_payment cap
  WHERE cap.coverage_month_start BETWEEN :period_start AND :period_end
),
by_plan AS (
  SELECT
      plan_npi, state_code, program_type,
      COUNT(*)                                              AS member_months,
      AVG(CASE WHEN encounters = 0 THEN 1.0 ELSE 0 END)     AS zero_encounter_rate,
      SUM(CASE WHEN encounters = 0 THEN paid_amount ELSE 0 END) AS paid_on_zero_encounter_months,
      COUNT(DISTINCT beneficiary_id)                        AS members
  FROM member_months
  GROUP BY plan_npi, state_code, program_type
  HAVING COUNT(*) >= :min_member_months
),
peer AS (
  SELECT state_code, program_type,
         AVG(zero_encounter_rate) AS peer_mean, STDDEV(zero_encounter_rate) AS peer_sd,
         COUNT(*) AS peer_n
  FROM by_plan GROUP BY state_code, program_type
)
SELECT
    p.plan_npi, p.state_code, p.program_type, p.members, p.member_months,
    ROUND(p.zero_encounter_rate, 3) AS zero_encounter_rate,
    ROUND(k.peer_mean, 3)           AS peer_mean,
    ROUND((p.zero_encounter_rate - k.peer_mean) / NULLIF(k.peer_sd, 0), 2) AS z_score,
    p.paid_on_zero_encounter_months
FROM by_plan p
JOIN peer k USING (state_code, program_type)
WHERE (p.zero_encounter_rate - k.peer_mean) / NULLIF(k.peer_sd, 0) > :z_threshold
ORDER BY p.paid_on_zero_encounter_months DESC;

-- Companion: members with 12 consecutive capitated months and zero encounters, by plan.
-- That population is worth an eligibility and existence check, not a utilisation review.
```

**4 · Threshold guidance.** :z_threshold = 2.5 against same-state, same-programme plans, :min_member_months = 1,000.
Population mix is the confound and it is substantial: a plan serving a younger, healthier
population will legitimately show more zero-encounter months, so compare within programme type
 -  a Medicaid expansion plan and a long-term-services plan are not peers. Do not read a zero-
encounter member as a fraud signal; read a zero-encounter *rate* against peers as an encounter-
data signal first, and only then as anything else. Encounter-data lag will also inflate the
most recent months, so exclude the last two months of the period. Tune to your population; a
starting point, not a standard.

**5 · Paired control + failure mode.** MD-17 (Managed-care encounter/payment data completeness monitoring + state corrective action). Secondary: MD-16 (T-MSIS submission + data-quality validation), MD-11 (State-directed payment & capitation actuarial-soundness oversight).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No encounter-completeness reconciliation exists. MD-17's reconciliation artifact does not exist. |
| **Designed-not-operating** | MD-17 is monthly; no reconciliation ran for the period. |
| **Operated-but-bypassed** | Completeness gaps were identified and no state corrective action followed, which is the second half of MD-17's design. |
| **Operated-but-ineffective** | Reconciliation counts encounter volume in aggregate and never measures member-level coverage, so a plan submitting for most members and none for some passes. |
| **Evidence-missing** | Reconciliation is performed by the plan and not independently evidenced by the state. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 13. Eligibility and enrollment data quality - impossible or missing attributes

| | |
|---|---|
| **Paired control** | **MD-16** - T-MSIS monthly submission + data-quality validation |
| **Control type** | Detective |
| **Module** | Module 07 - Medicaid & CHIP Integrity |
| **Owner** | T-MSIS / data unit |
| **Frequency** | Monthly |
| **System touchpoint** | T-MSIS pipeline + DQ tool |
| **Evidence artifact** | T-MSIS data-quality exception report |
| **Control failure mode** | Incomplete data blinds PI analytics |
| **Regulatory anchor** | CMS T-MSIS (Transformed MSIS) monthly data submission [guidance] |
| **Secondary controls** | MD-17 - Managed-care encounter/payment data completeness monitoring + state …; MD-12 - PERM / MEQC eligibility & claims sampling with error-rate measurement |

**1 · Data hypothesis.** Every other test in this family depends on eligibility data being right. Where T-MSIS data-
quality validation is absent or checks file structure rather than content, impossible values
persist - dates of birth in the future, eligibility spans ending before they start, missing
eligibility group codes, identifiers that collide. Bad eligibility data does not just hide
fraud; it manufactures false positives in every query above.

**2 · What to check.** Profile the eligibility and enrollment tables for structural impossibilities and missing
required attributes, quantify each defect class, and attach the paid dollars sitting behind
affected records so the data-quality argument carries a number.

**3 · Reference SQL.**

```sql
-- TEST 13 - Eligibility data-quality defect profile
-- Run this BEFORE tests 1-12. Their false-positive rates are a function of this output.
-- Compare against the CMS DQ Atlas measures: https://www.medicaid.gov/dq-atlas
-- Params: :period_start, :period_end

WITH defects_raw AS (
  SELECT b.beneficiary_id, 'DOB_IN_FUTURE' AS defect
  FROM beneficiary b WHERE b.date_of_birth > CURRENT_DATE
  UNION ALL
  SELECT b.beneficiary_id, 'DOB_IMPLAUSIBLE' FROM beneficiary b
   WHERE b.date_of_birth < CURRENT_DATE - INTERVAL '120 year'
  UNION ALL
  SELECT b.beneficiary_id, 'DOD_BEFORE_DOB' FROM beneficiary b
   WHERE b.date_of_death IS NOT NULL AND b.date_of_death < b.date_of_birth
  UNION ALL
  SELECT e.beneficiary_id, 'SPAN_ENDS_BEFORE_START' FROM eligibility_span e
   WHERE e.span_end IS NOT NULL AND e.span_end < e.span_start
  UNION ALL
  SELECT e.beneficiary_id, 'OVERLAPPING_SPANS' FROM eligibility_span e
   WHERE EXISTS (SELECT 1 FROM eligibility_span e2
                 WHERE e2.beneficiary_id = e.beneficiary_id
                   AND e2.span_id <> e.span_id
                   AND e.span_start <= COALESCE(e2.span_end, DATE '9999-12-31')
                   AND e2.span_start <= COALESCE(e.span_end, DATE '9999-12-31'))
  UNION ALL
  SELECT e.beneficiary_id, 'MISSING_ELIGIBILITY_GROUP' FROM eligibility_span e
   WHERE e.eligibility_group_code IS NULL
  UNION ALL
  SELECT b.beneficiary_id, 'DUPLICATE_IDENTITY_HASH' FROM beneficiary b
   WHERE EXISTS (SELECT 1 FROM beneficiary b2
                 WHERE b2.identity_hash = b.identity_hash
                   AND b2.beneficiary_id <> b.beneficiary_id)
),
defects AS (   -- one row per beneficiary and defect, so paid dollars are not counted twice
  SELECT DISTINCT beneficiary_id, defect FROM defects_raw
)
SELECT
    d.defect,
    COUNT(DISTINCT d.beneficiary_id) AS affected_beneficiaries,
    ROUND(COUNT(DISTINCT d.beneficiary_id) * 1.0 /
          NULLIF((SELECT COUNT(*) FROM beneficiary), 0), 5) AS defect_rate,
    COALESCE(SUM(x.paid_amount), 0) AS paid_behind_affected_records
FROM defects d
LEFT JOIN LATERAL (
    SELECT SUM(cl.paid_amount) AS paid_amount FROM claim_lines cl
    WHERE cl.beneficiary_id = d.beneficiary_id
      AND cl.service_date BETWEEN :period_start AND :period_end
      AND cl.adjudication_status = 'PAID'
) x ON TRUE
GROUP BY d.defect
ORDER BY paid_behind_affected_records DESC;
```

**4 · Threshold guidance.** Every defect class here is categorically wrong - a date of death before a date of birth is not
a tuning question. What varies is materiality, and `paid_behind_affected_records` is the column
that turns a data-quality complaint into a program-integrity argument. Run this test before any
other in the family: if OVERLAPPING_SPANS or DUPLICATE_IDENTITY_HASH is material, tests 3, 4
and 6 will produce large false-positive volumes that are attributable here rather than to any
provider or beneficiary. Compare your defect rates against the CMS DQ Atlas measures for the
same concepts so the numbers have external context. Tune to your population; a starting point,
not a standard.

**5 · Paired control + failure mode.** MD-16 (T-MSIS monthly submission + data-quality validation). Secondary: MD-17 (Managed-care encounter/payment data completeness monitoring), MD-12 (PERM / MEQC eligibility & claims sampling).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No data-quality validation operates. MD-16's T-MSIS data-quality exception report does not exist. |
| **Designed-not-operating** | MD-16 is monthly; validation did not run or produced no exception output. |
| **Operated-but-bypassed** | Exceptions were reported and no correction was made at source, so the same defects recur every month. |
| **Operated-but-ineffective** | Validation checks file structure and field format but not content plausibility, so structurally valid impossibilities pass. |
| **Evidence-missing** | Validation runs inside a vendor pipeline with no exception report surfaced to the state. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 14. Rapid disenrollment and agent-broker enrollment churn

| | |
|---|---|
| **Paired control** | **MC-16** - Agent/broker oversight: rapid-disenrollment & misrepresentation monitoring |
| **Control type** | Detective |
| **Module** | Module 06 - Managed Care (Part C/D) |
| **Owner** | Sales compliance |
| **Frequency** | Monthly |
| **System touchpoint** | Enrollment + complaint system |
| **Evidence artifact** | Agent oversight scorecard |
| **Control failure mode** | Marketing misrepresentation / churn fraud |
| **Regulatory anchor** | 42 CFR 422.2274 & 423.2274 (agent/broker & FDR oversight, rapid-disenrollment compensation recovery) [binding]; CMS Medicare Communications & Marketing Guidelines (MCMG) [guidance] |
| **Secondary controls** | PI-21 - Beneficiary-side integrity analytics - detect recruiter-clustered …; MD-09 - Concurrent multi-state MCO enrollment detection |

**1 · Data hypothesis.** Where agent and broker oversight is absent, enrollments driven by misrepresentation or
unauthorised switching produce a characteristic pattern: enrollments that reverse within a
short window, concentrated on particular agents, often with an associated complaint. Rapid
disenrollment is the measurable proxy for a beneficiary who did not understand or did not
consent to the enrollment.

**2 · What to check.** For each enrolling agent, compute the share of enrollments that disenroll within the rapid
window, benchmark against other agents, and join complaint volume. Report the compensation
exposure attached, since rapid-disenrollment compensation recovery is the mechanism the rule
provides.

**3 · Reference SQL.**

```sql
-- TEST 14 - Rapid-disenrollment rate by agent
-- 42 CFR 422.2274 / 423.2274 (agent-broker oversight, rapid-disenrollment compensation
-- recovery).
-- Params: :period_start, :period_end, :rapid_window_days (90), :min_enrollments (25), :z_threshold (2.5), :min_peer_n (5)

WITH enrollments AS (
  SELECT
      en.enrollment_id, en.agent_id, en.plan_npi, en.beneficiary_id,
      en.effective_date, en.disenroll_date,
      CASE WHEN en.disenroll_date IS NOT NULL
            AND en.disenroll_date - en.effective_date <= :rapid_window_days
           THEN 1 ELSE 0 END AS is_rapid,
      en.agent_compensation
  FROM enrollment en
  WHERE en.effective_date BETWEEN :period_start AND :period_end
),
by_agent AS (
  SELECT
      e.agent_id, a.agency_name, e.plan_npi,
      COUNT(*)                    AS enrollments,
      AVG(e.is_rapid)             AS rapid_rate,
      SUM(e.agent_compensation * e.is_rapid) AS compensation_on_rapid,
      (SELECT COUNT(*) FROM enrollment_complaint c
        WHERE c.agent_id = e.agent_id
          AND c.complaint_date BETWEEN :period_start AND :period_end) AS complaints
  FROM enrollments e
  LEFT JOIN agent a ON a.agent_id = e.agent_id
  GROUP BY e.agent_id, a.agency_name, e.plan_npi
),
peer AS (
  -- the volume floor applies to the rate comparison only, so the complaints signal below
  -- is surfaced independently of it
  SELECT plan_npi, AVG(rapid_rate) AS peer_mean, STDDEV(rapid_rate) AS peer_sd,
         COUNT(*) AS peer_n
  FROM by_agent WHERE enrollments >= :min_enrollments
  GROUP BY plan_npi HAVING COUNT(*) >= :min_peer_n
)
SELECT
    b.agent_id, b.agency_name, b.plan_npi, b.enrollments,
    ROUND(b.rapid_rate, 3) AS rapid_rate,
    ROUND(p.peer_mean, 3)  AS peer_mean,
    ROUND((b.rapid_rate - p.peer_mean) / NULLIF(p.peer_sd, 0), 2) AS z_score,
    b.complaints, b.compensation_on_rapid
FROM by_agent b
LEFT JOIN peer p USING (plan_npi)
WHERE (b.enrollments >= :min_enrollments
       AND (b.rapid_rate - p.peer_mean) / NULLIF(p.peer_sd, 0) > :z_threshold)
   OR b.complaints > 0
ORDER BY z_score DESC, b.compensation_on_rapid DESC;
```

**4 · Threshold guidance.** :rapid_window_days = 90 aligns with the conventional rapid-disenrollment definition, but
confirm against the applicable regulation and plan contract - the window drives the
compensation-recovery obligation, so getting it wrong has consequences beyond analytics.
:min_enrollments = 25 keeps the rate stable. Complaint volume is the more probative signal and
is surfaced independently of the z-score in the query above for that reason: a single
substantiated complaint about misrepresentation is worth more than a statistically elevated
rate, and a beneficiary who disenrolls because they moved or because a better plan appeared has
done nothing remarkable. Tune to your population; a starting point, not a standard.

**5 · Paired control + failure mode.** MC-16 (Agent/broker oversight: rapid-disenrollment & misrepresentation monitoring). Secondary: PI-21 (Beneficiary-side integrity analytics), MD-09 (Concurrent enrollment detection).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No agent oversight exists. MC-16's agent oversight scorecard does not exist. |
| **Designed-not-operating** | MC-16 is monthly; no scorecard was produced for the period. |
| **Operated-but-bypassed** | Outlier agents were identified and neither compensation recovery nor corrective action followed. |
| **Operated-but-ineffective** | The scorecard tracks enrollment volume and complaint counts separately and never joins them, so the agents with both never surface as a distinct group. |
| **Evidence-missing** | Monitoring occurs without a retained, dated scorecard tied to specific agents. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 15. Payment continued after a credible fraud allegation

| | |
|---|---|
| **Paired control** | **MD-18** - Credible-allegation-of-fraud payment suspension + MFCU referral |
| **Control type** | Corrective |
| **Module** | Module 07 - Medicaid & CHIP Integrity |
| **Owner** | Program-integrity office |
| **Frequency** | Per case |
| **System touchpoint** | Case management + suspension flag |
| **Evidence artifact** | Suspension notice + MFCU referral record |
| **Control failure mode** | Continued payment to fraudulent provider |
| **Regulatory anchor** | 42 CFR 455.23 (payment suspension on credible allegation) [binding]; 42 CFR Part 1007 (MFCU) [binding] |
| **Secondary controls** | PI-16 - Payment suspension on credible allegation of fraud; PI-14 - Suspected-fraud referral to OIG/MFCU/law enforcement |

**1 · Data hypothesis.** Where the payment-suspension control is absent or slow, payment continues to a provider after a
credible allegation of fraud has been determined - which is both an ongoing loss and a
regulatory failure under 42 CFR 455.23. The measurement is simple and rarely made: dollars paid
to a provider between the allegation determination date and the suspension effective date.

**2 · What to check.** For each credible-allegation determination, compute paid dollars in the interval before
suspension took effect, and identify determinations with no suspension and no documented good-
cause exception.

**3 · Reference SQL.**

```sql
-- TEST 15 - Payment between credible-allegation determination and suspension
-- 42 CFR 455.23. Good-cause exceptions to suspension are permitted and documented;
-- their absence, not their existence, is the finding.
-- Params: :period_start, :period_end

SELECT
    ca.allegation_id, ca.subject_npi, ca.determination_date,
    ca.referral_date, ca.referred_to,
    sus.suspension_effective_date,
    sus.good_cause_exception,
    CASE
      WHEN sus.good_cause_exception IS NOT NULL
        THEN 'GOOD_CAUSE_EXCEPTION'      -- documented and permitted: report it, do not call it a failure
      WHEN sus.suspension_id IS NULL OR sus.suspension_effective_date IS NULL
        THEN 'NO_SUSPENSION_NO_EXCEPTION'
      WHEN sus.suspension_effective_date > ca.determination_date
        THEN 'DELAYED_SUSPENSION'
      ELSE 'SUSPENDED'
    END AS suspension_status,
    COALESCE(sus.suspension_effective_date, CURRENT_DATE) - ca.determination_date
      AS days_paying_after_determination,
    (SELECT COALESCE(SUM(cl.paid_amount), 0) FROM claim_lines cl
      WHERE cl.billing_npi = ca.subject_npi
        AND cl.payment_date > ca.determination_date
        AND cl.payment_date < COALESCE(sus.suspension_effective_date, CURRENT_DATE)
        AND cl.adjudication_status = 'PAID') AS paid_after_determination
FROM credible_allegation ca
LEFT JOIN payment_suspension sus ON sus.allegation_id = ca.allegation_id
WHERE ca.determination_date BETWEEN :period_start AND :period_end
ORDER BY paid_after_determination DESC;

-- Also check the reverse direction for beneficiary protection: suspensions with no
-- MFCU referral, and suspensions running well past the point where the referral was
-- closed without action. A suspension that outlives its basis is its own failure.
```

**4 · Threshold guidance.** No statistical threshold - the numbers speak. `days_paying_after_determination` and
`paid_after_determination` are the two figures, and a good-cause exception under 455.23 is a
legitimate and documented reason for either, so the finding is specifically
NO_SUSPENSION_NO_EXCEPTION and unexplained DELAYED_SUSPENSION. Run the reverse check noted in
the query too: a suspension that continues after the referral it rests on has closed is a
failure in the other direction, affecting a provider's ability to operate on a basis that no
longer exists. Both directions belong in the same report. Tune to your population; a starting
point, not a standard.

**5 · Paired control + failure mode.** MD-18 (Credible-allegation-of-fraud payment suspension + MFCU referral). Secondary: PI-16 (Payment suspension on credible allegation of fraud), PI-14 (Suspected-fraud referral to OIG/MFCU/law enforcement).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No suspension capability exists. MD-18's suspension notice and MFCU referral record do not exist. |
| **Designed-not-operating** | The control is defined; determinations were made in the period and no suspension was actioned. |
| **Operated-but-bypassed** | Suspension was withheld without a documented good-cause exception, which 455.23 requires to be recorded. |
| **Operated-but-ineffective** | Suspension is applied to new claims only and does not stop payment on claims already in the adjudication pipeline, so a material sum pays after the effective date. |
| **Evidence-missing** | Suspensions occur but neither the determination basis nor the referral is retained in reviewable form. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 16. Services billed under a deceased or excluded provider identity

| | |
|---|---|
| **Paired control** | **PE-09** - State Medicaid exclusion / NPDB / death-master checks |
| **Control type** | Preventive |
| **Module** | Module 01 - Provider Enrollment |
| **Owner** | PI analyst |
| **Frequency** | Per app + periodic |
| **System touchpoint** | Multi-source screening |
| **Evidence artifact** | Multi-list screening log |
| **Control failure mode** | Cross-program excluded actors |
| **Regulatory anchor** | 42 CFR 455 Subpart E / 455.436 [binding]; OIG OEI-05-13-00520 (Medicaid Enhanced Provider Screenings) [case evidence] |
| **Secondary controls** | PE-08 - Monthly automated exclusion re-screening of active providers; PI-20 - Provider NPI/credential-footprint consistency monitoring - flag … |

**1 · Data hypothesis.** The provider-side mirror of test 1. Where multi-source screening - exclusion lists, NPDB, death
master file - is absent or applied only at enrollment, claims continue under the identity of a
provider who has died, been excluded, or had their licence revoked. A deceased provider's NPI
is a particularly clean signal, because unlike a beneficiary there is no legitimate post-death
claim.

**2 · What to check.** Join paid claims to the provider death-master and exclusion sources on both rendering and
billing NPI, flag service dates after death or exclusion, and measure the lag between the
screening source date and the last paid claim.

**3 · Reference SQL.**

```sql
-- TEST 16 - Claims under a deceased, excluded, or revoked provider identity
-- 42 CFR 455 Subpart E / 455.436. Params: :period_start, :period_end

WITH flagged_providers AS (
  SELECT npi, 'DECEASED'  AS flag_type, date_of_death AS flag_date, NULL AS reinstate_date
  FROM provider WHERE date_of_death IS NOT NULL
  UNION ALL
  SELECT npi, 'EXCLUDED',  exclusion_date, reinstate_date FROM exclusion_list
  UNION ALL
  SELECT npi, 'REVOKED',   revocation_date, reinstatement_date FROM licence_action
  WHERE action_type = 'REVOCATION'
)
SELECT
    cl.claim_line_id, cl.beneficiary_id, cl.service_date, cl.hcpcs, cl.paid_amount,
    cl.rendering_npi, cl.billing_npi,
    fp.npi AS flagged_npi, fp.flag_type, fp.flag_date,
    CASE WHEN fp.npi = cl.rendering_npi THEN 'RENDERING' ELSE 'BILLING' END AS flagged_role,
    (cl.service_date - fp.flag_date) AS days_after_flag
FROM claim_lines cl
JOIN flagged_providers fp
  ON fp.npi IN (cl.rendering_npi, cl.billing_npi)
WHERE cl.service_date > fp.flag_date
  AND (fp.reinstate_date IS NULL OR cl.service_date < fp.reinstate_date)
  AND cl.service_date BETWEEN :period_start AND :period_end
  AND cl.adjudication_status = 'PAID'
ORDER BY fp.flag_type, days_after_flag DESC, cl.paid_amount DESC;

-- The DECEASED bucket is the cleanest: unlike a beneficiary, there is no legitimate
-- post-death claim under a rendering provider's own NPI. Work it first regardless of value.
```

**4 · Threshold guidance.** Categorical, but the source data quality differs sharply by flag type. DECEASED matched on NPI
is high-confidence and needs no tuning. EXCLUDED matched on NPI is similarly reliable; matched
on name it is not, and needs identity resolution before action. REVOKED depends on state
licence-board feeds whose currency varies widely - check the feed's own lag before drawing
conclusions about a payer's screening. Screen against the source-list version effective for
each service date, not the current version, or you will manufacture retrospective findings for
claims that were correct when paid. Tune to your population; a starting point, not a standard.

**5 · Paired control + failure mode.** PE-09 (State Medicaid exclusion / NPDB / death-master checks). Secondary: PE-08 (Monthly automated exclusion re-screening of active providers), PI-20 (Provider NPI/credential-footprint consistency monitoring).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No multi-source screening operates. PE-09's multi-list screening log does not exist. |
| **Designed-not-operating** | PE-09 requires periodic screening beyond enrollment; no periodic run is evidenced for the period. |
| **Operated-but-bypassed** | Matches were identified and payment continued pending verification with no hold applied. |
| **Operated-but-ineffective** | Screening covers the exclusion list only and omits death-master and licence-action sources, so two of the three flag types are structurally invisible. |
| **Evidence-missing** | No retained record of which source-list versions were screened, or of negative results. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## Gap notes - controls this family needs that the library does not yet have

*Each note names a control objective these tests point to that the control library does not yet state. They are candidates for a future library release; the library itself is unchanged by this pack.*

### G1. No proof-of-service control exists for professional services - CD-09 is DMEPOS-scoped.

CD-09 is titled and written for proof of delivery: its trigger is 'delivery / service event',
its owner is 'supplier operations', its system is 'POD system', and its citation is 42 CFR
410.38. That is DMEPOS. Services-not-rendered recurs across the project's own case base
(personal care, hospice and home health, telemedicine and DME among others) and much of it
concerns professional and facility services, not durable medical equipment. Test 7 pairs to
CD-09 because there is nothing closer, and its 'Operated-but-ineffective' row records the
mismatch explicitly.

**Recommendation.** Recommended: a service-delivery validation control in Module 05, stating the objective
independently of the delivery mechanism - that an independent record evidences the encounter
occurred (EVV for in-home, appointment and check-in records for office, facility census for
institutional). This is the same gap family 1 raised as G1 for EVV, seen from the professional-
services side; the two should be resolved together rather than as separate controls, since they
share one objective.

### G2. No control addresses improper denial or wrongful disenrollment - the mirror risk of every test in this family.

The library has denial-appropriateness controls for prior authorisation - PA-14 (denial-
appropriateness sampling) and PA-15 (external review of upheld denials), both carrying open OIG
recommendations. It has no equivalent for *eligibility* decisions. Every test in this family
describes logic that, applied to a live determination rather than a paid claim, could terminate
coverage for an eligible person. The incoming work-requirements regime makes this sharper,
because improper denial or termination of an eligible person is both a program-integrity and a
beneficiary-protection failure, but the gap predates work requirements and applies to ordinary
redetermination, data-matching and ex-parte renewal today.

**Recommendation.** A beneficiary-protection control in Module 07, mirroring PA-14/PA-15 on the eligibility side:
sampling review of denials and terminations for appropriateness, false-positive disenrollment
rate monitoring, and notice-and-appeal adherence. This is arguably the single most
consequential gap the test pack has surfaced so far, because it is the one where the toolkit's
own logic, adopted uncritically, could cause the harm. Once the control exists the paired
analytics are straightforward - terminations reversed on appeal, by reason code and by office;
ex-parte renewal failure rates against peer states.

### G3. Tests 1 and 2 supersede family 1's tests 5 and 6.

Family 1 (home-health time integrity) included deceased-beneficiary and incarceration tests
scoped to in-home services. Tests 1 and 2 here generalise both across all service types and add
the capitation leg, which is the larger exposure. This is not a defect - family 1 needed them
in context - but a payer running both packs should not run four tests where two suffice.

**Recommendation.** Note in any consolidated view that family 4 tests 1-2 are the canonical versions and family 1
tests 5-6 are the home-health-scoped subset. If the pack is ever flattened into a single
crosswalk sheet, de-duplicate in that direction. No change to either published family is
proposed.

### G4. Work-requirements (community engagement) eligibility has no controls, so no analytics can be paired yet.

CMS-2454-IFC requires certain adults to meet 80 hours/month of qualifying activity as a
condition of eligibility, with state implementation no later than 1 January 2027. This creates
eligibility-integrity surfaces this family would otherwise cover: hours-verification data
integrity, exemption-determination correctness, and the wrongful-termination mirror risk. The
control library has nothing on it, so there is nothing to pair a query to.

**Recommendation.** Known gap: the work-requirements controls are still to be scoped and built. Family 10 of this
test pack is reserved for newly-built control families; when the work-requirements controls
land, the analytics should be added there rather than retrofitted here. Recorded here so the
dependency is visible in both directions.

---

*Reference patterns. Public-source, personal-capacity, advisory and organizational only. Queries must be adapted to your own schema and tested on your own warehouse before use. Outputs are review candidates for human adjudication, never payment or denial dispositions. Confirm every regulatory anchor against its primary source.*