# Analytics Test Pack - Home-health / attendant-care time integrity

*impossible-day, overlapping hours, and billed-vs-verified visit time*

*Generated 2026-11-11 from the HPI-CT control library (`data/controls.csv`, 173 controls). 16 detection tests, paired to 15 primary controls. Reference patterns, not programs.*

## Before you run anything

**These queries produce review candidates, never payment or denial decisions.** Every hit is a question for a human.

**The toolkit never receives your data.** These are patterns you run inside your own environment. No claim data, no PHI, no identifiers leave your systems.

**Every query is a pattern, not a program.** Field names are generic; yours will differ. Expect to rewrite the FROM and JOIN clauses and to remap service categories to your own taxonomy.

**Every threshold is a starting point, not a standard.** Where a threshold is arithmetic it is labelled as such. Everywhere else, measure your own distribution first.

Read `PREFACE_how_to_run_these.md` before the first run.

## Scope of this family

Personal care, attendant care, home-health aide, self-directed/consumer-directed care, HCBS
waiver in-home services, private-duty nursing, and adult day services - any service billed in
time units where the unit of production is a human being's hours. Personal care and home-based
services are a recurring scheme in the case base (34 counted matters in the personal care
category and 66 in hospice and home health, after the 2026-10 re-screening), and this is the
family where the data signature of a broken control is clearest: hours that could not
physically have been worked.

Every test in this family reduces to arithmetic on time - minutes per person per day, intervals
that overlap, distance that cannot be travelled in the gap between two visits. That makes the
queries cheap to run, easy to explain to a provider, and hard to argue with. It also makes them
the best first demonstration of the detection→diagnosis loop: a hit here points at a small,
named set of controls.

## Tests at a glance

| # | Test | Primary control | Also touches |
|---|---|---|---|
| 1 | Impossible day - >24 billed hours for one rendering provider | **CB-08** | CB-07, MD-14 |
| 2 | Overlapping visit intervals - one aide in two places at once | **CB-08** | MD-14, CD-09 |
| 3 | Geographic impossibility - travel time between consecutive visits | **PI-20** | CB-08 |
| 4 | Billed-vs-verified variance - claims with no EVV visit record | **CD-09** | MD-24, CD-03 |
| 5 | Service billed after date of death | **MD-08** | MD-06 |
| 6 | In-home services billed during incarceration or an institutional stay | **MD-22** | CD-17, MD-06 |
| 7 | Excluded aide or agency with paid claims after the exclusion date | **PE-08** | PE-07, MD-04 |
| 8 | Authorized-vs-billed hours overage against the service plan | **MD-14** | CB-08, CD-10 |
| 9 | Duplicate time units across programs, payers, or agencies | **CB-15** | CD-17 |
| 10 | Representative-approver conflict in self-directed care | **MD-24** | PI-21 |
| 11 | Attendant–beneficiary roster overlap and recruiter clustering | **PI-21** | PI-20, MD-24 |
| 12 | Uniform-duration visits - the templated-timesheet signature | **CD-07** | CD-06, CD-09 |
| 13 | Missing or invalid plan-of-care order behind ongoing in-home services | **CD-08** | CD-10, CD-03 |
| 14 | Agency-level hours-per-beneficiary peer outlier | **PI-01** | MD-14, PI-02 |
| 15 | New-agency billing velocity in the first 90 days | **PE-15** | PE-16, PI-02 |
| 16 | Recertification lapse on continuing in-home services | **CD-10** | CD-03, MD-14 |

> **Control-ID validation:** every control referenced above resolves to a real ID in the current control library. See the gap notes at the end for controls this family *needs* but the library does not yet have.

---

## 1. Impossible day - >24 billed hours for one rendering provider

| | |
|---|---|
| **Paired control** | **CB-08** - Quantity / frequency-limit edits (capped & time-based services) |
| **Control type** | Detective |
| **Module** | Module 03 - Coding & Billing Integrity |
| **Owner** | PI analyst / edit builder |
| **Frequency** | Continuous |
| **System touchpoint** | Claims analytics + edits |
| **Evidence artifact** | Frequency-limit exception report |
| **Control failure mode** | Overpayment for CCM / over-limit items |
| **Regulatory anchor** | CMS NCCI Medically Unlikely Edits (MUE) [guidance]; OIG A-07-19-05122 (6 Aug 2021) - Chronic Care Management overpayments; edit rec 21-A-07-129.03 closed unimplemented, notification rec .02 open [case evidence] |
| **Secondary controls** | CB-07 - MUE units-of-service edit enforcement; MD-14 - Utilization control & SURS medical-review program |

**1 · Data hypothesis.** If the quantity/frequency-limit edit is absent or tuned only at the claim-line level, an aide
or agency can bill time units that sum across claims to more hours than exist in a day.
Fraudulent and erroneous time-billing both leave the same signature: a per-NPI per-day minute
total that no human schedule supports. Line-level MUE edits do not catch it because each
individual line is plausible; only the daily aggregate is impossible.

**2 · What to check.** For each rendering provider (aide/nurse) and each service date, sum all billed service minutes
across every claim line in the time-based service categories. Flag any provider-day above a
physical-plausibility ceiling. Run a second, tighter pass at a sustained-workload ceiling (e.g.
>16h) to catch the more common and more defensible-looking pattern.

**3 · Reference SQL.**

```sql
-- TEST 1 - Impossible day: total billed service minutes per rendering NPI per day
-- Pattern only. Run on your own warehouse. Output = review candidates, not dispositions.
-- Params: :period_start, :period_end, :minutes_per_unit (15), :hard_ceiling_minutes (1440),
--         :soft_ceiling_minutes (960)

WITH timed_lines AS (
  SELECT
      cl.rendering_npi,
      cl.service_date,
      -- normalise units to minutes; adapt the unit->minute factor to YOUR fee schedule.
      -- Many personal-care HCPCS bill in 15-minute units (T1019, S5125, G0156).
      SUM(COALESCE(cl.service_minutes, cl.units * :minutes_per_unit)) AS billed_minutes,
      COUNT(*)                                   AS line_count,
      COUNT(DISTINCT cl.beneficiary_id)          AS distinct_beneficiaries,
      COUNT(DISTINCT cl.billing_npi)             AS distinct_billing_entities,
      SUM(cl.paid_amount)                        AS paid_amount
  FROM claim_lines cl
  WHERE cl.service_date BETWEEN :period_start AND :period_end
    AND cl.service_category IN ('personal_care','attendant_care','home_health_aide',
                                'private_duty_nursing','hcbs_in_home')   -- tune to your taxonomy
    AND cl.adjudication_status = 'PAID'
    AND cl.rendering_npi IS NOT NULL
  GROUP BY cl.rendering_npi, cl.service_date
)
SELECT
    rendering_npi,
    service_date,
    billed_minutes,
    ROUND(billed_minutes / 60.0, 2) AS billed_hours,
    line_count,
    distinct_beneficiaries,
    distinct_billing_entities,
    paid_amount,
    CASE
      WHEN billed_minutes > :hard_ceiling_minutes THEN 'IMPOSSIBLE'   -- >24h in a calendar day
      ELSE 'IMPLAUSIBLE'                                              -- above sustained-workload ceiling
    END AS severity
FROM timed_lines
WHERE billed_minutes > :soft_ceiling_minutes
ORDER BY billed_minutes DESC, paid_amount DESC;

-- Companion roll-up: providers with a RECURRING impossible-day pattern are the
-- stronger review candidates than a single-day outlier (which is often a keying error).
--   SELECT rendering_npi, COUNT(*) AS impossible_days, SUM(paid_amount) AS exposure
--   FROM ( <query above> ) x WHERE severity = 'IMPOSSIBLE'
--   GROUP BY rendering_npi HAVING COUNT(*) >= :recurrence_threshold;
```

**4 · Threshold guidance.** :hard_ceiling_minutes = 1440 (24h) is a physical fact, not a tuned threshold - a hit above it
is an arithmetic impossibility and should always be reviewed. :soft_ceiling_minutes = 960 (16h)
is the tunable one: start there and move it against your own population, because live-in and
24-hour-care programs, split-shift authorisations, and multi-payer aides legitimately produce
long days. :minutes_per_unit is normally 15 for T-code personal care but MUST be read off your
fee schedule per HCPCS. :recurrence_threshold = 3 impossible days in the period is a reasonable
starting filter. Tune to your population; a starting point, not a standard.

**5 · Paired control + failure mode.** CB-08 (Quantity / frequency-limit edits - capped & time-based services). Secondary: CB-07 (MUE units-of-service edit enforcement), MD-14 (Utilization control & SURS medical-review program).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No daily-aggregate frequency edit exists - only line-level MUE. Build the per-NPI-per-day rollup edit; CB-08's evidence artifact (frequency-limit exception report) does not exist. |
| **Designed-not-operating** | The exception report is specified but was not produced for the period. Restore the continuous cadence and name the owner (PI analyst / edit builder). |
| **Operated-but-bypassed** | Impossible-day exceptions were produced and closed with no action or a blanket override. Require a written rationale per closure. |
| **Operated-but-ineffective** | The edit runs at claim-line level only, so it never sees the daily aggregate - the rule is the defect, not the operation. Add the cross-claim daily rollup. |
| **Evidence-missing** | Staff assert the review happens but no dated, attributable exception log is retained. The control may be sound; the assurance is not. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 2. Overlapping visit intervals - one aide in two places at once

| | |
|---|---|
| **Paired control** | **CB-08** - Quantity / frequency-limit edits (capped & time-based services) |
| **Control type** | Detective |
| **Module** | Module 03 - Coding & Billing Integrity |
| **Owner** | PI analyst / edit builder |
| **Frequency** | Continuous |
| **System touchpoint** | Claims analytics + edits |
| **Evidence artifact** | Frequency-limit exception report |
| **Control failure mode** | Overpayment for CCM / over-limit items |
| **Regulatory anchor** | CMS NCCI Medically Unlikely Edits (MUE) [guidance]; OIG A-07-19-05122 (6 Aug 2021) - Chronic Care Management overpayments; edit rec 21-A-07-129.03 closed unimplemented, notification rec .02 open [case evidence] |
| **Secondary controls** | MD-14 - Utilization control & SURS medical-review program; CD-09 - Proof-of-delivery / proof-of-service capture and retention with all … |

**1 · Data hypothesis.** The daily-total test misses the case where an aide bills 8 plausible hours that happen to be
the SAME 8 hours for two different beneficiaries. If the time-integrity edit has no interval-
overlap rule, concurrent service intervals for one rendering provider pass adjudication
cleanly. This is the single most durable signature of attendant-care fraud in the case base.

**2 · What to check.** Self-join billed service intervals for the same rendering provider on the same date and flag
any pair whose [start, end) windows intersect for more than a de-minimis tolerance. Report the
overlap minutes, both beneficiaries, and both billing entities - cross-agency overlaps are a
distinct and stronger signal.

**3 · Reference SQL.**

```sql
-- TEST 2 - Overlapping billed service intervals for one rendering provider
-- Requires start/end timestamps. If you only hold date+units, this test is not
-- runnable on claims alone - run it against EVV check-in/check-out instead (see TEST 8).
-- Params: :period_start, :period_end, :overlap_tolerance_minutes (0-5)

WITH intervals AS (
  SELECT
      cl.claim_line_id,
      cl.rendering_npi,
      cl.billing_npi,
      cl.beneficiary_id,
      cl.service_date,
      cl.service_begin_datetime AS t_start,
      cl.service_end_datetime   AS t_end,
      cl.paid_amount
  FROM claim_lines cl
  WHERE cl.service_date BETWEEN :period_start AND :period_end
    AND cl.service_category IN ('personal_care','attendant_care','home_health_aide',
                                'private_duty_nursing','hcbs_in_home')
    AND cl.adjudication_status = 'PAID'
    AND cl.service_begin_datetime IS NOT NULL
    AND cl.service_end_datetime   IS NOT NULL
)
SELECT
    a.rendering_npi,
    a.service_date,
    a.claim_line_id       AS line_a,
    b.claim_line_id       AS line_b,
    a.beneficiary_id      AS beneficiary_a,
    b.beneficiary_id      AS beneficiary_b,
    a.billing_npi         AS billing_entity_a,
    b.billing_npi         AS billing_entity_b,
    a.t_start AS a_start, a.t_end AS a_end,
    b.t_start AS b_start, b.t_end AS b_end,
    -- overlap in minutes between the two intervals
    (LEAST(a.t_end, b.t_end) - GREATEST(a.t_start, b.t_start)) AS overlap_window,
    (a.paid_amount + b.paid_amount) AS combined_paid,
    CASE WHEN a.billing_npi <> b.billing_npi THEN 'CROSS-AGENCY' ELSE 'SAME-AGENCY' END AS overlap_class
FROM intervals a
JOIN intervals b
  ON  a.rendering_npi = b.rendering_npi
  AND a.service_date  = b.service_date
  AND a.claim_line_id < b.claim_line_id          -- each pair once, no self-match
  AND a.t_start < b.t_end
  AND b.t_start < a.t_end                        -- classic interval-intersection test
WHERE a.beneficiary_id <> b.beneficiary_id       -- drop same-beneficiary split shifts
  -- the overlap must be longer than the tolerance (0 keeps every overlap)
  AND (LEAST(a.t_end, b.t_end) - GREATEST(a.t_start, b.t_start))
        > :overlap_tolerance_minutes * INTERVAL '1' MINUTE
ORDER BY a.rendering_npi, a.service_date;

-- NOTE ON DIALECT: timestamp subtraction returns an interval in Postgres, a number of
-- days in Oracle, and requires DATEDIFF/TIMESTAMPDIFF in SQL Server / BigQuery / Snowflake.
-- Replace the LEAST/GREATEST expression with your dialect's minute-difference function.
-- Portability: number * INTERVAL '1' MINUTE is PostgreSQL/DuckDB style; the portable form of the
-- tolerance test is DATEDIFF/TIMESTAMPDIFF(MINUTE, overlap_start, overlap_end) > :overlap_tolerance_minutes.
```

**4 · Threshold guidance.** There is no statistical threshold here - an overlap is either present or it is not. The tunable
is :overlap_tolerance_minutes: allow 0–5 minutes to absorb rounding and clock skew; the query
returns only pairs whose overlap is longer than that tolerance. Prioritise CROSS-AGENCY
overlaps and providers with ≥5 overlapping pairs in the period; a single 3-minute same-agency
overlap is almost always a documentation artifact. Tune to your population; a starting point,
not a standard.

**5 · Paired control + failure mode.** CB-08 (Quantity / frequency-limit edits - capped & time-based services). Secondary: MD-14 (Utilization control & SURS), CD-09 (Proof-of-delivery / proof-of-service capture).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No interval-overlap rule exists anywhere in the edit set. Build it - this is the highest-yield single rule in the family. |
| **Designed-not-operating** | The overlap monitor is designed but has not been run against the period's claims. |
| **Operated-but-bypassed** | Overlap exceptions were dispositioned as 'documentation error' without correcting the paid claim or referring the pattern. |
| **Operated-but-ineffective** | The rule compares dates but not times, or runs within a single agency only - so cross-agency and same-day overlaps are invisible. |
| **Evidence-missing** | No retained exception log ties an overlap hit to a named reviewer and a dated disposition. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 3. Geographic impossibility - travel time between consecutive visits

| | |
|---|---|
| **Paired control** | **PI-20** - Provider NPI/credential-footprint consistency monitoring - flag claims where a billing NPI's service location, billing TIN, or claim geography is inconsistent with the provider's enrollment/known practice footprint |
| **Control type** | Detective |
| **Module** | Module 02 - Program Integrity |
| **Owner** | SIU / PI data analyst |
| **Frequency** | Continuous / near-real-time |
| **System touchpoint** | Claims + enrollment (PECOS/MMIS) join |
| **Evidence artifact** | Footprint-inconsistency alert log + dispositions |
| **Control failure mode** | Legitimately-enrolled NPI billed by an unrelated party for services the named provider never rendered (pattern from a 2025 state enforcement operation) |
| **Regulatory anchor** | 42 CFR 455 Subpart A (Medicaid Agency Fraud Detection & Investigation Program) incl. §455.23 (payment suspension on credible fraud allegation) [binding]; §455 Subpart E (provider screening) [binding] |
| **Secondary controls** | CB-08 - Quantity / frequency-limit edits (capped & time-based services) |

**1 · Data hypothesis.** Where the NPI/credential-footprint consistency monitor is absent or checks only the enrolled
service address, an aide can bill sequential non-overlapping visits that are geographically
unreachable in the gap between them. Non-overlapping intervals defeat TEST 2; distance defeats
the claim.

**2 · What to check.** Order each rendering provider's visits by start time within a date, compute the gap to the next
visit and the straight-line distance between the two service locations, and flag pairs where
the implied travel speed exceeds a plausible ceiling. Straight-line distance under-states real
travel, so a violation on straight-line distance is conservative.

**3 · Reference SQL.**

```sql
-- TEST 3 - Geographic impossibility between consecutive visits
-- Uses service-location coordinates (EVV geo-stamp, or geocoded beneficiary service address).
-- If you hold no coordinates, degrade to ZIP-centroid distance and raise the speed ceiling.
-- Params: :period_start, :period_end, :max_speed_mph (65), :min_distance_miles (5)
-- Portability: date +/- integer days and date - date (a day count) are PostgreSQL/DuckDB style;
--   elsewhere use DATEADD / DATE_ADD for the offset and DATEDIFF / DATE_DIFF for the difference.
-- Portability: EXTRACT(EPOCH FROM a - b) is PostgreSQL/DuckDB style; elsewhere use DATEDIFF or
--   TIMESTAMPDIFF in seconds.

WITH visits AS (
  SELECT
      cl.rendering_npi,
      cl.service_date,
      cl.beneficiary_id,
      cl.claim_line_id,
      cl.service_begin_datetime AS t_start,
      cl.service_end_datetime   AS t_end,
      loc.latitude,
      loc.longitude
  FROM claim_lines cl
  JOIN service_location loc                -- your geocoded service-address dimension
    ON loc.beneficiary_id = cl.beneficiary_id
  WHERE cl.service_date BETWEEN :period_start AND :period_end
    AND cl.service_category IN ('personal_care','attendant_care','home_health_aide',
                                'private_duty_nursing','hcbs_in_home')
    AND cl.adjudication_status = 'PAID'
),
sequenced AS (
  SELECT
      v.*,
      LEAD(t_start)       OVER w AS next_start,
      LEAD(latitude)      OVER w AS next_lat,
      LEAD(longitude)     OVER w AS next_lon,
      LEAD(beneficiary_id) OVER w AS next_beneficiary
  FROM visits v
  WINDOW w AS (PARTITION BY rendering_npi, service_date ORDER BY t_start)
),
legs AS (
  SELECT
      rendering_npi, service_date, beneficiary_id, next_beneficiary,
      t_end, next_start,
      -- haversine, miles. Replace with ST_DISTANCE if your warehouse has geospatial types.
      3958.8 * 2 * ASIN(SQRT(
          POWER(SIN(RADIANS(next_lat - latitude) / 2), 2)
        + COS(RADIANS(latitude)) * COS(RADIANS(next_lat))
        * POWER(SIN(RADIANS(next_lon - longitude) / 2), 2)
      )) AS distance_miles,
      -- gap in hours between end of one visit and start of the next
      EXTRACT(EPOCH FROM (next_start - t_end)) / 3600.0 AS gap_hours
  FROM sequenced
  WHERE next_start IS NOT NULL
)
SELECT
    rendering_npi, service_date,
    beneficiary_id AS from_beneficiary, next_beneficiary AS to_beneficiary,
    t_end AS left_at, next_start AS arrived_at,
    ROUND(distance_miles, 1) AS distance_miles,
    ROUND(gap_hours, 2)      AS gap_hours,
    ROUND(distance_miles / NULLIF(gap_hours, 0), 1) AS implied_mph
FROM legs
WHERE distance_miles > :min_distance_miles
  AND distance_miles / NULLIF(gap_hours, 0) > :max_speed_mph
ORDER BY implied_mph DESC;
```

**4 · Threshold guidance.** :max_speed_mph = 65 is a defensible urban/suburban starting ceiling; raise it in rural states
with long interstate runs and lower it in dense metros where 65mph door-to-door is itself
implausible. :min_distance_miles = 5 suppresses the noise from geocoding imprecision on
adjacent addresses. Zero gap_hours (back-to-back visits at different addresses) is the
strongest sub-case - surface it separately. Tune to your population; a starting point, not a
standard.

**5 · Paired control + failure mode.** PI-20 (Provider NPI/credential-footprint consistency monitoring). Secondary: CB-08 (Quantity / frequency-limit edits).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No footprint-consistency monitor exists; PI-20's alert log does not exist. |
| **Designed-not-operating** | PI-20 is specified as continuous/near-real-time but produced no alerts in the period. |
| **Operated-but-bypassed** | Geographic alerts were cleared as 'aide relocated' with no corroborating record. |
| **Operated-but-ineffective** | The monitor compares the billing NPI's enrolled address only, never the sequence of service locations within a day. |
| **Evidence-missing** | Alerts fire into a queue with no retained disposition record. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 4. Billed-vs-verified variance - claims with no EVV visit record

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
| **Secondary controls** | MD-24 - Self-directed-care approver-independence control - where a …; CD-03 - Reasonable-and-necessary documentation control: the medical record … |

**1 · Data hypothesis.** Where the proof-of-service capture control is absent, not reconciled, or reconciled only in one
direction, claims exist for visits that no electronic visit verification record supports.
Services-not-rendered leaves its cleanest signature as a claim with no matching verified visit
 -  the payer holds both datasets and simply never joins them.

**2 · What to check.** Left-join paid time-based claim lines to the EVV visit table on beneficiary + attendant + date,
and report (a) claims with no EVV visit at all, and (b) claims whose billed minutes materially
exceed the EVV-verified minutes. Run the reciprocal direction too - verified visits never
billed indicate a data pipeline problem rather than fraud, and distinguishing the two is the
point.

**3 · Reference SQL.**

```sql
-- TEST 4 - Billed-vs-EVV-verified variance (services-not-rendered signature)
-- EVV is required for Medicaid personal care and home health services under
-- 21st Century Cures Act §12006 (SSA §1903(l)). Confirm your state's EVV scope.
-- Params: :period_start, :period_end, :minutes_per_unit (15), :variance_tolerance_pct (0.10)

WITH billed AS (
  SELECT
      cl.claim_line_id, cl.beneficiary_id, cl.rendering_npi, cl.billing_npi,
      cl.service_date, cl.hcpcs, cl.paid_amount,
      COALESCE(cl.service_minutes, cl.units * :minutes_per_unit) AS billed_minutes
  FROM claim_lines cl
  WHERE cl.service_date BETWEEN :period_start AND :period_end
    AND cl.service_category IN ('personal_care','attendant_care','home_health_aide','hcbs_in_home')
    AND cl.adjudication_status = 'PAID'
),
verified AS (
  SELECT
      v.beneficiary_id,
      v.attendant_npi,
      CAST(v.check_in_datetime AS DATE) AS service_date,
      SUM(v.verified_minutes)           AS verified_minutes,
      COUNT(*)                          AS verified_visits
  FROM evv_visit v
  WHERE v.visit_status = 'COMPLETE'
    AND CAST(v.check_in_datetime AS DATE) BETWEEN :period_start AND :period_end
  GROUP BY v.beneficiary_id, v.attendant_npi, CAST(v.check_in_datetime AS DATE)
)
SELECT
    b.claim_line_id, b.beneficiary_id, b.rendering_npi, b.billing_npi,
    b.service_date, b.hcpcs, b.billed_minutes,
    COALESCE(v.verified_minutes, 0) AS verified_minutes,
    b.billed_minutes - COALESCE(v.verified_minutes, 0) AS excess_minutes,
    b.paid_amount,
    CASE
      WHEN v.beneficiary_id IS NULL THEN 'NO_EVV_RECORD'
      WHEN b.billed_minutes > v.verified_minutes * (1 + :variance_tolerance_pct)
           THEN 'BILLED_EXCEEDS_VERIFIED'
      ELSE 'WITHIN_TOLERANCE'
    END AS variance_class
FROM billed b
LEFT JOIN verified v
       ON  v.beneficiary_id = b.beneficiary_id
       AND v.attendant_npi  = b.rendering_npi
       AND v.service_date   = b.service_date
WHERE v.beneficiary_id IS NULL
   OR b.billed_minutes > v.verified_minutes * (1 + :variance_tolerance_pct)
ORDER BY excess_minutes DESC, b.paid_amount DESC;

-- Reciprocal pass (data-integrity, not fraud): verified visits with no matching claim.
-- A high reciprocal rate means your EVV-to-claims key is broken and the forward
-- direction above will over-report. Run the reciprocal FIRST and clear it.
```

**4 · Threshold guidance.** :variance_tolerance_pct = 0.10 absorbs rounding between EVV minutes and billed units; set it to
0 once your unit-to-minute mapping is proven. Before treating NO_EVV_RECORD as a finding,
establish the match rate on the reciprocal pass - a match rate below ~95% means the join key
(not the provider) is the problem. Prioritise agencies where NO_EVV_RECORD exceeds ~5% of paid
lines. Tune to your population; a starting point, not a standard.

**5 · Paired control + failure mode.** CD-09 (Proof-of-delivery / proof-of-service capture and retention). Secondary: MD-24 (Self-directed-care approver-independence - carries the EVV / Cures Act §12006 anchor), CD-03 (Reasonable-and-necessary documentation).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No billed-vs-delivered reconciliation exists between EVV and claims. Build it - see the GAP NOTE below on the missing dedicated EVV-reconciliation control. |
| **Designed-not-operating** | The variance report is defined but was not produced for the period. |
| **Operated-but-bypassed** | Variances were resolved by accepting an after-the-fact manual visit entry with no independent corroboration. |
| **Operated-but-ineffective** | Reconciliation runs at aggregate agency level and nets overs against unders, so line-level phantom visits vanish in the average. |
| **Evidence-missing** | The reconciliation is asserted but no dated variance report or disposition is retained. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 5. Service billed after date of death

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
| **Secondary controls** | MD-06 - Enrollee eligibility verification via authoritative electronic data … |

**1 · Data hypothesis.** If the Death Master File match is absent, run late, or matched only against eligibility rather
than against paid claims, in-home services continue to be billed for a beneficiary who has
died. The signature is unambiguous: a service date after the recorded date of death.

**2 · What to check.** Join paid time-based claim lines to the beneficiary date-of-death and flag any service date
strictly after it. Report separately the claims dated after death but SUBMITTED before the
death was recorded (a data-latency finding) and those submitted after (a stronger candidate).

**3 · Reference SQL.**

```sql
-- TEST 5 - Service dates after date of death
-- Params: :period_start, :period_end, :grace_days (0)
-- Portability: date +/- integer days and date - date (a day count) are PostgreSQL/DuckDB style;
--   elsewhere use DATEADD / DATE_ADD for the offset and DATEDIFF / DATE_DIFF for the difference.

SELECT
    cl.claim_id, cl.claim_line_id, cl.beneficiary_id,
    cl.billing_npi, cl.rendering_npi, cl.hcpcs,
    cl.service_date, b.date_of_death,
    (cl.service_date - b.date_of_death) AS days_after_death,
    cl.units, cl.paid_amount,
    cl.claim_submission_date,
    CASE
      WHEN cl.claim_submission_date > b.death_record_load_date
        THEN 'SUBMITTED_AFTER_DEATH_KNOWN'      -- stronger review candidate
      ELSE 'SUBMITTED_BEFORE_DEATH_KNOWN'        -- likely data-latency; still recoverable
    END AS submission_class
FROM claim_lines cl
JOIN beneficiary b
  ON b.beneficiary_id = cl.beneficiary_id
WHERE b.date_of_death IS NOT NULL
  AND cl.service_date > b.date_of_death + :grace_days
  AND cl.service_date BETWEEN :period_start AND :period_end
  AND cl.adjudication_status = 'PAID'
ORDER BY days_after_death DESC, cl.paid_amount DESC;

-- Companion: the same test against CAPITATION payments, not just FFS claim lines,
-- which is where the larger dollar exposure usually sits.
```

**4 · Threshold guidance.** :grace_days = 0 for in-home services - unlike facility claims there is no legitimate post-death
service. Some warehouses carry an imprecise date of death from a secondary source; if yours
does, set :grace_days = 1 and treat same-day-as-death claims as a separate, lower-priority
bucket rather than suppressing them. Escalate any provider with ≥3 post-death claims. Tune to
your population; a starting point, not a standard.

**5 · Paired control + failure mode.** MD-08 (Deceased-enrollee detection via SSA Death Master File match). Secondary: MD-06 (Enrollee eligibility verification via authoritative electronic data sources).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No DMF match runs. MD-08's monthly exception disposition does not exist. |
| **Designed-not-operating** | MD-08 is specified monthly but the match did not run for one or more months in the period - establish what was exposed in the gap. |
| **Operated-but-bypassed** | DMF matches were dispositioned as false positives in bulk without a documented identity-resolution step. |
| **Operated-but-ineffective** | The match updates eligibility but never sweeps already-paid claims, so it prevents future capitation without recovering paid FFS. |
| **Evidence-missing** | No retained match output or dated disposition per exception. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 6. In-home services billed during incarceration or an institutional stay

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
| **Secondary controls** | CD-17 - Coordination-of-benefits / setting documentation control to prevent …; MD-06 - Enrollee eligibility verification via authoritative electronic data … |

**1 · Data hypothesis.** In-home attendant care cannot be furnished to someone who is in jail, in an inpatient bed, or
in a nursing facility. Where the incarceration/institutional status match is absent or runs
only against eligibility, home-care claims overlap institutional custody dates cleanly.

**2 · What to check.** Interval-join paid in-home claim lines against (a) the corrections roster and (b)
inpatient/SNF/ICF institutional stays, and flag any service date falling inside a custody or
admission window. The institutional-stay half of this test is usually higher-volume and
entirely internal to the payer's own claims.

**3 · Reference SQL.**

```sql
-- TEST 6 - In-home service dates inside an incarceration or institutional-stay window
-- Params: :period_start, :period_end
-- Note CAA 2024 requires SUSPEND-not-terminate for incarcerated enrollees (eff. 1/1/2026);
-- the control objective is suspension of PAYMENT, not loss of eligibility.

-- 6a: overlap with corrections custody
SELECT
    'INCARCERATION' AS conflict_type,
    cl.claim_line_id, cl.beneficiary_id, cl.billing_npi, cl.rendering_npi,
    cl.service_date, cl.hcpcs, cl.units, cl.paid_amount,
    inc.book_date, inc.release_date, inc.facility_id AS conflict_facility
FROM claim_lines cl
JOIN incarceration_roster inc
  ON  inc.beneficiary_id = cl.beneficiary_id
  AND cl.service_date >= inc.book_date
  AND cl.service_date <= COALESCE(inc.release_date, DATE '9999-12-31')
WHERE cl.service_date BETWEEN :period_start AND :period_end
  AND cl.service_category IN ('personal_care','attendant_care','home_health_aide','hcbs_in_home')
  AND cl.adjudication_status = 'PAID'

UNION ALL

-- 6b: overlap with an inpatient / SNF / ICF stay (entirely internal to your claims)
SELECT
    'INSTITUTIONAL_STAY' AS conflict_type,
    cl.claim_line_id, cl.beneficiary_id, cl.billing_npi, cl.rendering_npi,
    cl.service_date, cl.hcpcs, cl.units, cl.paid_amount,
    st.admit_date AS book_date, st.discharge_date AS release_date, st.facility_type
FROM claim_lines cl
JOIN institutional_stay st
  ON  st.beneficiary_id = cl.beneficiary_id
  -- exclude the admit and discharge days themselves: partial-day home care is legitimate
  AND cl.service_date >  st.admit_date
  AND cl.service_date <  COALESCE(st.discharge_date, DATE '9999-12-31')
WHERE cl.service_date BETWEEN :period_start AND :period_end
  AND cl.service_category IN ('personal_care','attendant_care','home_health_aide','hcbs_in_home')
  AND cl.adjudication_status = 'PAID'
ORDER BY paid_amount DESC;
```

**4 · Threshold guidance.** No statistical threshold - a conflict is categorical. The tunables are the boundary-day rules:
EXCLUDE admit and discharge days (partial-day home care around a hospitalisation is legitimate
and will otherwise generate large false-positive volume), and consider excluding the booking
day for incarceration. Rank by paid_amount and by conflict days per beneficiary; a single
boundary-adjacent day is noise, a 30-day run is not. Tune to your population; a starting point,
not a standard.

**5 · Paired control + failure mode.** MD-22 (Incarceration/institutional-status match - suspend claims/capitation during incarceration). Secondary: CD-17 (Coordination-of-benefits / setting documentation control), MD-06 (Enrollee eligibility verification).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No corrections-data feed and no cross-setting conflict edit. MD-22's monthly exception report does not exist. |
| **Designed-not-operating** | MD-22 is specified monthly; the roster match did not run, or ran without the paid-claims join it specifies. |
| **Operated-but-bypassed** | Conflicts were closed on an unverified assertion that the enrollee was released early. |
| **Operated-but-ineffective** | The match suspends future capitation only and never tests already-paid in-home FFS claims - which is where this scheme lives. |
| **Evidence-missing** | Match runs but no dated exception report with dispositions is retained. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 7. Excluded aide or agency with paid claims after the exclusion date

| | |
|---|---|
| **Paired control** | **PE-08** - Monthly automated exclusion re-screening of active providers |
| **Control type** | Detective |
| **Module** | Module 01 - Provider Enrollment |
| **Owner** | Compliance/PI |
| **Frequency** | Monthly, aligned to OIG LEIE release (~10th; file covers actions through prior month) |
| **System touchpoint** | Exclusion-screening tool |
| **Evidence artifact** | Dated LEIE file version used + active-roster snapshot screened + match output (retain 'no matches' result); confirmed matches remediated <=5 business days; retain >=6 yrs |
| **Control failure mode** | Post-enrollment exclusion not caught |
| **Regulatory anchor** | OIG Compliance Program Guidance (2023) [guidance] |
| **Secondary controls** | PE-07 - Exclusion screening at enrollment (LEIE + SAM); MD-04 - Federal-database screening of providers/owners (LEIE, SAM, Death … |

**1 · Data hypothesis.** If monthly exclusion re-screening is absent, run against a stale LEIE file, or screened against
the enrolled provider roster but not the rendering-provider field on claims, an excluded
individual continues to render and be paid for in-home services. Home care is the highest-risk
setting for this because the rendering party is an individual aide, often not separately
enrolled.

**2 · What to check.** Join paid claim lines to the exclusion list on both billing and rendering NPI (and, where you
hold it, on the attendant roster identifier) and flag any service date on or after the
exclusion date and before any reinstatement date.

**3 · Reference SQL.**

```sql
-- TEST 7 - Paid claims rendered or billed by an excluded party
-- Screen BOTH the billing entity and the rendering individual. In home care the
-- rendering individual is the usual gap.
-- Params: :period_start, :period_end
-- Portability: date +/- integer days and date - date (a day count) are PostgreSQL/DuckDB style;
--   elsewhere use DATEADD / DATE_ADD for the offset and DATEDIFF / DATE_DIFF for the difference.

WITH excluded AS (
  SELECT npi, exclusion_date, reinstate_date, exclusion_type, source_file_date
  FROM exclusion_list                       -- OIG LEIE + GSA SAM + state exclusion lists
  WHERE npi IS NOT NULL
)
SELECT
    cl.claim_line_id, cl.beneficiary_id, cl.service_date, cl.hcpcs, cl.paid_amount,
    cl.billing_npi, cl.rendering_npi,
    e.npi            AS excluded_npi,
    CASE WHEN e.npi = cl.rendering_npi THEN 'RENDERING' ELSE 'BILLING' END AS excluded_role,
    e.exclusion_date, e.reinstate_date, e.exclusion_type,
    (cl.service_date - e.exclusion_date) AS days_after_exclusion
FROM claim_lines cl
JOIN excluded e
  ON e.npi IN (cl.rendering_npi, cl.billing_npi)
WHERE cl.service_date >= e.exclusion_date
  AND (e.reinstate_date IS NULL OR cl.service_date < e.reinstate_date)
  AND cl.service_date BETWEEN :period_start AND :period_end
  AND cl.adjudication_status = 'PAID'
ORDER BY days_after_exclusion DESC, cl.paid_amount DESC;

-- Companion (the harder, more valuable pass): match the ATTENDANT ROSTER by
-- name + DOB-hash against LEIE for aides who hold no NPI. Do this inside your own
-- environment; it is identity matching and needs a documented resolution step.
```

**4 · Threshold guidance.** Categorical, not statistical - but exclusion matching by name is noisy, so treat NPI-keyed
matches as high-confidence and name-keyed roster matches as candidates requiring identity
resolution before any action. Screen against the LEIE file version dated closest to (and not
after) each service date; PE-08 specifies monthly alignment to the ~10th-of-month LEIE release.
Tune to your population; a starting point, not a standard.

**5 · Paired control + failure mode.** PE-08 (Monthly automated exclusion re-screening of active providers). Secondary: PE-07 (Exclusion screening at enrollment), MD-04 (Federal-database screening of providers/owners).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No monthly re-screen. PE-08's dated LEIE-version + roster-snapshot evidence does not exist. |
| **Designed-not-operating** | Re-screen skipped one or more months; PE-08's evidence artifact names the file version precisely so gaps are provable. |
| **Operated-but-bypassed** | A match was identified and the provider continued to be paid pending 'verification' with no payment hold. |
| **Operated-but-ineffective** | The screen covers enrolled billing entities only and never touches the rendering/attendant population - the actual exposure in home care. |
| **Evidence-missing** | No retained record of the LEIE file version used or the roster snapshot screened. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 8. Authorized-vs-billed hours overage against the service plan

| | |
|---|---|
| **Paired control** | **MD-14** - Utilization control & SURS medical-review program |
| **Control type** | Detective |
| **Module** | Module 07 - Medicaid & CHIP Integrity |
| **Owner** | Medical review / SURS unit |
| **Frequency** | Continuous |
| **System touchpoint** | SURS analytics |
| **Evidence artifact** | Medical-review case dispositions |
| **Control failure mode** | Medically unnecessary / aberrant utilization |
| **Regulatory anchor** | 42 CFR Part 456 (utilization control/SURS) [binding]; CMS State Medicaid Manual [guidance] |
| **Secondary controls** | CB-08 - Quantity / frequency-limit edits (capped & time-based services); CD-10 - Continued-need / recertification documentation control for ongoing … |

**1 · Data hypothesis.** Where utilization control is absent or reviews medical necessity without comparing delivered
units to the authorized service plan, billed hours drift above the authorization. Sustained
overage against a person-centred service plan is both an overpayment signature and a plan-
integrity signal.

**2 · What to check.** Aggregate billed units per beneficiary per HCPCS per week and compare to the authorized units
for that period from the service plan / prior authorization. Report the overage ratio and the
number of consecutive weeks in overage.

**3 · Reference SQL.**

```sql
-- TEST 8 - Billed units above the authorized service-plan units
-- Params: :period_start, :period_end, :overage_tolerance_pct (0.05)
-- Portability: DATE_TRUNC is PostgreSQL/Snowflake/DuckDB style; elsewhere use DATETRUNC (SQL Server),
--   TRUNC(date, 'MM') (Oracle) or DATE_TRUNC(date, MONTH) (BigQuery).

WITH billed_weekly AS (
  SELECT
      cl.beneficiary_id,
      cl.hcpcs,
      cl.billing_npi,
      DATE_TRUNC('week', cl.service_date) AS service_week,
      SUM(cl.units)       AS billed_units,
      SUM(cl.paid_amount) AS paid_amount
  FROM claim_lines cl
  WHERE cl.service_date BETWEEN :period_start AND :period_end
    AND cl.service_category IN ('personal_care','attendant_care','home_health_aide','hcbs_in_home')
    AND cl.adjudication_status = 'PAID'
  GROUP BY cl.beneficiary_id, cl.hcpcs, cl.billing_npi, DATE_TRUNC('week', cl.service_date)
),
authorized AS (
  SELECT
      sp.beneficiary_id, sp.hcpcs,
      sp.authorized_units_per_week,
      sp.effective_start, sp.effective_end
  FROM service_plan sp                      -- or your prior_auth table
  WHERE sp.status = 'ACTIVE'
)
SELECT
    b.beneficiary_id, b.hcpcs, b.billing_npi, b.service_week,
    b.billed_units,
    a.authorized_units_per_week,
    b.billed_units - a.authorized_units_per_week AS overage_units,
    ROUND(b.billed_units / NULLIF(a.authorized_units_per_week, 0), 2) AS overage_ratio,
    b.paid_amount,
    CASE WHEN a.beneficiary_id IS NULL THEN 'NO_AUTHORIZATION_ON_FILE' ELSE 'OVER_AUTHORIZATION' END
      AS overage_class
FROM billed_weekly b
LEFT JOIN authorized a
       ON  a.beneficiary_id = b.beneficiary_id
       AND a.hcpcs          = b.hcpcs
       AND b.service_week BETWEEN a.effective_start AND COALESCE(a.effective_end, DATE '9999-12-31')
WHERE a.beneficiary_id IS NULL
   OR b.billed_units > a.authorized_units_per_week * (1 + :overage_tolerance_pct)
ORDER BY overage_ratio DESC NULLS LAST, b.paid_amount DESC;
```

**4 · Threshold guidance.** :overage_tolerance_pct = 0.05 absorbs legitimate week-boundary spill (a shift spanning midnight
Sunday). NO_AUTHORIZATION_ON_FILE is usually the larger and more actionable bucket and often
indicates a data-linkage break rather than fraud - separate it before reporting. Prioritise
beneficiaries with ≥4 consecutive overage weeks or an overage_ratio ≥1.25. Tune to your
population; a starting point, not a standard.

**5 · Paired control + failure mode.** MD-14 (Utilization control & SURS medical-review program). Secondary: CB-08 (Quantity / frequency-limit edits), CD-10 (Continued-need / recertification documentation control).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No authorization-to-delivery comparison exists in the utilization-control program. |
| **Designed-not-operating** | MD-14 is specified continuous but produced no case dispositions for the period. |
| **Operated-but-bypassed** | Overages were retroactively authorized without a documented reassessment of need. |
| **Operated-but-ineffective** | Review is triggered by cost thresholds only, so a low-rate service in chronic overage never surfaces. |
| **Evidence-missing** | Medical-review case dispositions are not retained in a form that ties back to the specific overage. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 9. Duplicate time units across programs, payers, or agencies

| | |
|---|---|
| **Paired control** | **CB-15** - Duplicate claim / line detection |
| **Control type** | Detective |
| **Module** | Module 03 - Coding & Billing Integrity |
| **Owner** | Billing system owner |
| **Frequency** | Per claim |
| **System touchpoint** | Claim edit engine |
| **Evidence artifact** | Duplicate-hit disposition log |
| **Control failure mode** | Duplicate payment for one service |
| **Regulatory anchor** | OIG A-01-20-00503 (22 Nov 2022) - 6 of 100 sampled services billed twice by two providers on the same day [case evidence]; duplicate-service edit rec (.03) closed, co-surgery education rec (.04) open [case evidence]; OIG A-06-21-05003 (19 Sep 2022) - duplicate professional services; edit rec 22-A-06-098.04 closed, recovery recs .01 and .02 open [case evidence] |
| **Secondary controls** | CD-17 - Coordination-of-benefits / setting documentation control to prevent … |

**1 · Data hypothesis.** If duplicate detection keys only on (claim, line, provider) it will not catch the same
beneficiary-hour billed by two different agencies, or under two different programs (waiver vs
state plan), or to two different payers. The duplicate is real; the key is wrong.

**2 · What to check.** Group paid time-based lines by beneficiary + service date + service window and flag any group
containing lines from more than one billing entity, program, or payer where the combined units
exceed what a single beneficiary could receive.

**3 · Reference SQL.**

```sql
-- TEST 9 - Cross-agency / cross-program duplicate time units for one beneficiary-day
-- Params: :period_start, :period_end, :minutes_per_unit (15), :max_beneficiary_minutes_per_day (1440)

WITH bene_day AS (
  SELECT
      cl.beneficiary_id,
      cl.service_date,
      SUM(COALESCE(cl.service_minutes, cl.units * :minutes_per_unit)) AS total_minutes,
      COUNT(DISTINCT cl.billing_npi)   AS distinct_agencies,
      COUNT(DISTINCT cl.payer_program) AS distinct_programs,
      COUNT(DISTINCT cl.rendering_npi) AS distinct_aides,
      SUM(cl.paid_amount)              AS paid_amount
  FROM claim_lines cl
  WHERE cl.service_date BETWEEN :period_start AND :period_end
    AND cl.service_category IN ('personal_care','attendant_care','home_health_aide','hcbs_in_home')
    AND cl.adjudication_status = 'PAID'
  GROUP BY cl.beneficiary_id, cl.service_date
)
SELECT
    beneficiary_id, service_date,
    total_minutes,
    ROUND(total_minutes / 60.0, 2) AS total_hours,
    distinct_agencies, distinct_programs, distinct_aides, paid_amount,
    CASE
      WHEN total_minutes > :max_beneficiary_minutes_per_day THEN 'EXCEEDS_24H_OF_CARE'
      WHEN distinct_agencies > 1 THEN 'MULTI_AGENCY_SAME_DAY'
      ELSE 'MULTI_PROGRAM_SAME_DAY'
    END AS duplicate_class
FROM bene_day
WHERE total_minutes > :max_beneficiary_minutes_per_day
   OR distinct_agencies > 1
   OR distinct_programs > 1
ORDER BY total_minutes DESC, paid_amount DESC;

-- Tighter companion: exact-duplicate line pairs across agencies - 
-- same beneficiary_id, service_date, hcpcs and units, different billing_npi.
```

**4 · Threshold guidance.** EXCEEDS_24H_OF_CARE is arithmetic and always reviewable. MULTI_AGENCY_SAME_DAY is legitimately
common (a nurse visit plus an aide shift) and is a triage signal, not a finding - rank it by
total_minutes and only escalate above roughly the 95th percentile of your own beneficiary-day
minute distribution. Compute that percentile first; do not import a number. Tune to your
population; a starting point, not a standard.

**5 · Paired control + failure mode.** CB-15 (Duplicate claim / line detection). Secondary: CD-17 (Coordination-of-benefits / setting documentation control).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No duplicate detection beyond exact claim resubmission. |
| **Designed-not-operating** | CB-15's duplicate-hit disposition log is empty for the period. |
| **Operated-but-bypassed** | Duplicate hits were overridden at adjudication to clear a backlog. |
| **Operated-but-ineffective** | The duplicate key includes billing_npi, so cross-agency duplicates can never match. This is the defining defect for this test. |
| **Evidence-missing** | Hits are suppressed silently at the edit engine with no retained disposition log. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 10. Representative-approver conflict in self-directed care

| | |
|---|---|
| **Paired control** | **MD-24** - Self-directed-care approver-independence control - where a conservator/guardian/authorized representative approves attendant time, require an independent secondary check + relationship/conflict screen (shared address/family/account between representative and attendant) before payment |
| **Control type** | Preventive |
| **Module** | Module 07 - Medicaid & CHIP Integrity |
| **Owner** | State HCBS PI / MCO |
| **Frequency** | Continuous screen + quarterly sample |
| **System touchpoint** | EVV / claims + representative & attendant registries |
| **Evidence artifact** | Conflict-screen result + representative-approved-claim sampling disposition |
| **Control failure mode** | Representative-approver with a financial tie to the caregiver approving fabricated time (pattern from a 2026 state self-directed-care case) |
| **Regulatory anchor** | 42 CFR 441 Subpart G (HCBS §1915(c) waivers) & Subpart K (Community First Choice) [binding]; 21st Century Cures Act §12006 → SSA §1903(l) (EVV mandate) [binding] |
| **Secondary controls** | PI-21 - Beneficiary-side integrity analytics - detect recruiter-clustered … |

**1 · Data hypothesis.** In self-directed/consumer-directed programs the person approving the attendant's timesheet is
often the beneficiary's representative. Where the approver-independence control is absent, a
representative with a financial or household tie to the attendant approves their own
household's hours. The data signature is a representative-attendant pair sharing address or
contact details, or a representative approving an implausible concentration of hours.

**2 · What to check.** Join the representative registry to the attendant registry on hashed address and hashed phone,
and separately rank representatives by total approved hours and by the share of a single
attendant's hours they approve. Do the identity comparison on hashed values inside your own
environment.

**3 · Reference SQL.**

```sql
-- TEST 10 - Self-directed care: representative-approver conflict
-- Identity comparison uses HASHED values, computed and compared inside your own
-- environment. No identifiers leave the payer.
-- Params: :period_start, :period_end, :min_conflict_signals (1 for triage, 2 for a prioritised queue)

WITH approvals AS (
  SELECT
      ts.representative_id,
      ts.attendant_id,
      ts.beneficiary_id,
      SUM(ts.approved_minutes) AS approved_minutes,
      COUNT(*)                 AS approved_timesheets,
      SUM(ts.paid_amount)      AS paid_amount
  FROM timesheet_approval ts
  WHERE ts.approval_date BETWEEN :period_start AND :period_end
    AND ts.program_type = 'SELF_DIRECTED'
  GROUP BY ts.representative_id, ts.attendant_id, ts.beneficiary_id
),
conflicts AS (
  SELECT
      a.*,
      CASE WHEN r.address_hash = at.address_hash THEN 1 ELSE 0 END AS shared_address,
      CASE WHEN r.phone_hash   = at.phone_hash   THEN 1 ELSE 0 END AS shared_phone,
      CASE WHEN r.surname_hash = at.surname_hash THEN 1 ELSE 0 END AS shared_surname
  FROM approvals a
  JOIN representative r ON r.representative_id = a.representative_id
  JOIN attendant      at ON at.attendant_id    = a.attendant_id
)
SELECT
    representative_id, attendant_id, beneficiary_id,
    approved_minutes, ROUND(approved_minutes / 60.0, 2) AS approved_hours,
    approved_timesheets, paid_amount,
    shared_address, shared_phone, shared_surname,
    (shared_address + shared_phone + shared_surname) AS conflict_signal_count
FROM conflicts
WHERE (shared_address + shared_phone + shared_surname) >= :min_conflict_signals
ORDER BY conflict_signal_count DESC, paid_amount DESC;

-- Companion, no registry join required: representatives in the top percentile of
-- approved hours, or approving >:concentration_pct of one attendant's total hours.
```

**4 · Threshold guidance.** :min_conflict_signals = 1 for triage, 2 for a prioritised queue. Be careful: a shared address
between representative and attendant is EXPECTED and lawful in many family-caregiver programs  - 
this test identifies where the approver-independence control should have been APPLIED, not
where fraud occurred. That distinction belongs in the output header. :concentration_pct = 0.90
of one attendant's hours approved by one representative is a reasonable companion cut. Tune to
your population; a starting point, not a standard.

**5 · Paired control + failure mode.** MD-24 (Self-directed-care approver-independence control). Secondary: PI-21 (Beneficiary-side integrity analytics - roster-overlap detection).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No conflict screen between representative and attendant registries exists. |
| **Designed-not-operating** | MD-24 specifies a continuous screen plus quarterly sampling; neither ran. |
| **Operated-but-bypassed** | Conflicts were identified and waived without the independent second-approver MD-24 requires. |
| **Operated-but-ineffective** | The screen matches exact name strings only, so it misses hashed-address and phone ties and any name variation. |
| **Evidence-missing** | No retained conflict-screen result or sampling disposition. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 11. Attendant–beneficiary roster overlap and recruiter clustering

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
| **Secondary controls** | PI-20 - Provider NPI/credential-footprint consistency monitoring - flag …; MD-24 - Self-directed-care approver-independence control - where a … |

**1 · Data hypothesis.** Where beneficiary-side integrity analytics are absent, schemes that pay beneficiaries to
participate leave two signatures: individuals appearing on both the beneficiary roll and the
staff/attendant roster, and beneficiaries whose enrollments cluster tightly around one
recruiter, address, or referring entity.

**2 · What to check.** Intersect the attendant roster with the beneficiary roll on hashed identity, and separately
count beneficiaries per shared residential address hash and per enrolling/referring entity,
flagging clusters above the population norm.

**3 · Reference SQL.**

```sql
-- TEST 11 - Roster overlap + address/recruiter clustering
-- Hashed identity comparison inside your own environment only.
-- Params: :period_start, :period_end, :cluster_threshold

-- 11a: individuals on BOTH the attendant roster and the beneficiary roll
SELECT
    'ROSTER_OVERLAP' AS signal,
    at.attendant_id,
    b.beneficiary_id,
    at.employer_billing_npi,
    COUNT(cl.claim_line_id) AS paid_lines,
    SUM(cl.paid_amount)     AS paid_amount
FROM attendant at
JOIN beneficiary b
  ON  b.identity_hash = at.identity_hash        -- e.g. sha256(lower(name)||dob||ssn_last4)
LEFT JOIN claim_lines cl
  ON  cl.rendering_npi = at.attendant_npi
  AND cl.service_date BETWEEN :period_start AND :period_end
  AND cl.adjudication_status = 'PAID'
GROUP BY at.attendant_id, b.beneficiary_id, at.employer_billing_npi

UNION ALL

-- 11b: beneficiaries clustered at one residential address served by one agency
SELECT
    'ADDRESS_CLUSTER' AS signal,
    CAST(NULL AS VARCHAR)      AS attendant_id,
    CAST(NULL AS VARCHAR)      AS beneficiary_id,
    cl.billing_npi             AS employer_billing_npi,
    COUNT(DISTINCT cl.beneficiary_id) AS paid_lines,
    SUM(cl.paid_amount)               AS paid_amount
FROM claim_lines cl
JOIN service_location loc ON loc.beneficiary_id = cl.beneficiary_id
WHERE cl.service_date BETWEEN :period_start AND :period_end
  AND cl.service_category IN ('personal_care','attendant_care','home_health_aide','hcbs_in_home')
  AND cl.adjudication_status = 'PAID'
GROUP BY loc.address_hash, cl.billing_npi
HAVING COUNT(DISTINCT cl.beneficiary_id) >= :cluster_threshold
ORDER BY paid_amount DESC;
```

**4 · Threshold guidance.** :cluster_threshold = 5 distinct beneficiaries at one address hash served by one agency is a
starting cut - but group homes, assisted-living settings and large apartment buildings will
dominate the output, so exclude known congregate addresses FIRST or the test is unusable. For
roster overlap, any hit is worth identity resolution; the volume is small. Compute your own
distribution of beneficiaries-per-address before fixing the threshold. Tune to your population;
a starting point, not a standard.

**5 · Paired control + failure mode.** PI-21 (Beneficiary-side integrity analytics). Secondary: PI-20 (Provider NPI/credential-footprint consistency monitoring), MD-24 (Self-directed-care approver-independence).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No enrollment+claims+HR-roster join exists. PI-21's recruiter-cluster / roster-overlap exception report does not exist. |
| **Designed-not-operating** | PI-21 is specified monthly; the report was not produced. |
| **Operated-but-bypassed** | Overlaps were dismissed as coincidence with no identity-resolution step recorded. |
| **Operated-but-ineffective** | The analytic runs on claims only and never joins the HR/attendant roster, so the defining signal is unavailable to it. |
| **Evidence-missing** | No retained exception report or dispositions. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 12. Uniform-duration visits - the templated-timesheet signature

| | |
|---|---|
| **Paired control** | **CD-07** - Copy-forward / templated-note control: EHR authorship controls limit cloned documentation so each note reflects the specific encounter |
| **Control type** | Detective |
| **Module** | Module 05 - Coverage & Documentation |
| **Owner** | HIM / Compliance |
| **Frequency** | Quarterly |
| **System touchpoint** | EHR audit log |
| **Evidence artifact** | Cloned-note audit result |
| **Control failure mode** | Cloned/templated notes that do not support the billed service |
| **Regulatory anchor** | 42 CFR 424.5(a)(6) (documentation must support services) [binding]; Medicare Program Integrity Manual (Pub 100-08) ch.3 [guidance]; OIG guidance on cloned/copy-paste documentation [guidance] |
| **Secondary controls** | CD-06 - Signature legibility & attestation control: legible identifiable …; CD-09 - Proof-of-delivery / proof-of-service capture and retention with all … |

**1 · Data hypothesis.** Real care visits vary in length. Where the copy-forward/templated-documentation control is
absent or applies to clinical notes but not to time entry, manufactured timesheets show near-
zero variance: every visit exactly the same duration, often starting on the same clock minute.
Low variance is not proof of anything, but it is a cheap and effective prioritisation signal.

**2 · What to check.** For each rendering provider with enough volume, compute the standard deviation and the modal
share of billed visit durations, and the share of visits starting at an exact quarter-hour.
Rank providers by lowest duration variance.

**3 · Reference SQL.**

```sql
-- TEST 12 - Uniform-duration / templated-timesheet signature
-- Prioritisation signal only. A low-variance provider is a review candidate, never a finding.
-- Params: :period_start, :period_end, :minutes_per_unit (15), :min_visits (30),
--         :max_coefficient_of_variation (0.05), :max_quarter_start_share (0.95)

WITH visit_durations AS (
  SELECT
      cl.rendering_npi,
      cl.billing_npi,
      cl.claim_line_id,
      COALESCE(cl.service_minutes, cl.units * :minutes_per_unit) AS duration_minutes,
      EXTRACT(MINUTE FROM cl.service_begin_datetime)             AS start_minute
  FROM claim_lines cl
  WHERE cl.service_date BETWEEN :period_start AND :period_end
    AND cl.service_category IN ('personal_care','attendant_care','home_health_aide','hcbs_in_home')
    AND cl.adjudication_status = 'PAID'
),
provider_stats AS (
  SELECT
      rendering_npi,
      billing_npi,
      COUNT(*)                       AS visit_count,
      AVG(duration_minutes)          AS mean_minutes,
      STDDEV(duration_minutes)       AS sd_minutes,
      AVG(CASE WHEN start_minute IN (0, 15, 30, 45) THEN 1.0 ELSE 0 END) AS exact_quarter_start_share
  FROM visit_durations
  GROUP BY rendering_npi, billing_npi
  HAVING COUNT(*) >= :min_visits
)
SELECT
    rendering_npi, billing_npi, visit_count,
    ROUND(mean_minutes, 1) AS mean_minutes,
    ROUND(sd_minutes, 2)   AS sd_minutes,
    ROUND(sd_minutes / NULLIF(mean_minutes, 0), 4) AS coefficient_of_variation,
    ROUND(exact_quarter_start_share, 3) AS exact_quarter_start_share
FROM provider_stats
WHERE sd_minutes / NULLIF(mean_minutes, 0) < :max_coefficient_of_variation
   OR exact_quarter_start_share > :max_quarter_start_share
ORDER BY coefficient_of_variation ASC;
```

**4 · Threshold guidance.** :min_visits = 30 keeps the statistic meaningful. :max_coefficient_of_variation = 0.05 (i.e.
duration SD under 5% of the mean) is a starting cut, but note that authorised fixed-length
shifts legitimately produce near-zero variance - so this test's value depends entirely on
excluding fixed-shift authorisations first. :max_quarter_start_share = 0.95 is a second,
independent signal. Use both together; either alone is weak. Tune to your population; a
starting point, not a standard.

**5 · Paired control + failure mode.** CD-07 (Copy-forward / templated-note control). Secondary: CD-06 (Signature legibility & attestation control), CD-09 (Proof-of-delivery / proof-of-service capture).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No cloned-documentation control covers time entry at all. |
| **Designed-not-operating** | CD-07 specifies a quarterly cloned-note audit; it was not performed in the period. |
| **Operated-but-bypassed** | The audit identified templated entries and no coding/documentation corrective action followed. |
| **Operated-but-ineffective** | The control examines narrative notes only and never inspects the time fields, which is where the manufacture happens. |
| **Evidence-missing** | No retained cloned-note audit result. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 13. Missing or invalid plan-of-care order behind ongoing in-home services

| | |
|---|---|
| **Paired control** | **CD-08** - Valid-order verification: a treating physician/practitioner order is on file before the service (signed where the rule for the item or service requires it); order authentication enforced; ordering-physician IDENTITY/NPI authenticated at the authoritative source (NPPES/PECOS) - guards against stolen/forged real-clinician identities |
| **Control type** | Preventive |
| **Module** | Module 05 - Coverage & Documentation |
| **Owner** | Intake / Scheduling |
| **Frequency** | Per service |
| **System touchpoint** | EHR / order management + NPPES/PECOS |
| **Evidence artifact** | Signed order on file + source-verified ordering-provider identity |
| **Control failure mode** | Service furnished without a valid order, or under a stolen/forged clinician identity (42 CFR 410.32) |
| **Regulatory anchor** | Medicare Benefit Policy Manual (CMS Pub 100-02) ch.15 sec.80.6 (orders for diagnostic tests) [guidance]; 42 CFR 410.32(a) and (d)(2) (diagnostic tests ordered by the treating physician; documentation kept) [binding]; 42 CFR 410.38(d)(1) (DMEPOS written order/prescription) [binding] |
| **Secondary controls** | CD-10 - Continued-need / recertification documentation control for ongoing …; CD-03 - Reasonable-and-necessary documentation control: the medical record … |

**1 · Data hypothesis.** Home-health and HCBS services require a signed order/plan of care from an enrolled
practitioner. Where valid-order verification is absent or checks only that a field is
populated, services are furnished against a missing order, an expired plan of care, or an
ordering NPI that is not enrolled, is deceased, or is excluded.

**2 · What to check.** For each paid in-home claim line, confirm an active signed order exists covering the service
date and that the ordering NPI resolves to an enrolled, non-excluded, living practitioner.
Report each failure reason separately - they have different remediations.

**3 · Reference SQL.**

```sql
-- TEST 13 - Order integrity behind ongoing in-home services
-- Params: :period_start, :period_end

WITH matched_order AS (
  -- for each claim line, pick one order that began on or before the service date: an order
  -- whose effective window covers the service date is preferred (latest started first);
  -- only when no order covers the date does the latest-started order stand in, so that the
  -- CASE below labels the line ORDER_EXPIRED rather than NO_ORDER_ON_FILE.
  SELECT cl.claim_line_id, o.order_id, o.signature_date, o.effective_start, o.effective_end,
         ROW_NUMBER() OVER (PARTITION BY cl.claim_line_id
                            ORDER BY CASE WHEN cl.service_date <= COALESCE(o.effective_end, DATE '9999-12-31')
                                          THEN 0 ELSE 1 END,
                                     o.effective_start DESC) AS rn
  FROM claim_lines cl
  JOIN service_order o
    ON  o.beneficiary_id = cl.beneficiary_id
    AND o.hcpcs_group    = cl.hcpcs_group
    AND o.effective_start <= cl.service_date
  WHERE cl.service_date BETWEEN :period_start AND :period_end
)
SELECT
    cl.claim_line_id, cl.beneficiary_id, cl.billing_npi, cl.ordering_npi,
    cl.service_date, cl.hcpcs, cl.paid_amount,
    o.order_id, o.signature_date, o.effective_start, o.effective_end,
    p.enrollment_status, p.date_of_death AS ordering_provider_dod,
    CASE
      WHEN o.order_id IS NULL                                   THEN 'NO_ORDER_ON_FILE'
      WHEN o.signature_date IS NULL                             THEN 'ORDER_UNSIGNED'
      WHEN cl.service_date > COALESCE(o.effective_end, DATE '9999-12-31')
                                                                THEN 'ORDER_EXPIRED'
      WHEN cl.service_date < o.signature_date                   THEN 'SERVICE_BEFORE_SIGNATURE'
      WHEN p.npi IS NULL                                        THEN 'ORDERING_NPI_NOT_FOUND'
      WHEN p.enrollment_status <> 'ACTIVE'                      THEN 'ORDERING_NPI_NOT_ENROLLED'
      WHEN p.date_of_death IS NOT NULL
           AND cl.service_date > p.date_of_death                THEN 'ORDERING_PROVIDER_DECEASED'
      ELSE 'OK'
    END AS order_defect
FROM claim_lines cl
LEFT JOIN matched_order o ON o.claim_line_id = cl.claim_line_id AND o.rn = 1
LEFT JOIN provider p ON p.npi = cl.ordering_npi
WHERE cl.service_date BETWEEN :period_start AND :period_end
  AND cl.service_category IN ('personal_care','attendant_care','home_health_aide',
                              'private_duty_nursing','hcbs_in_home')
  AND cl.adjudication_status = 'PAID'
  AND (o.order_id IS NULL
       OR o.signature_date IS NULL
       OR cl.service_date > COALESCE(o.effective_end, DATE '9999-12-31')
       OR cl.service_date < o.signature_date
       OR p.npi IS NULL
       OR p.enrollment_status <> 'ACTIVE'
       OR (p.date_of_death IS NOT NULL AND cl.service_date > p.date_of_death))
ORDER BY order_defect, cl.paid_amount DESC;
```

**4 · Threshold guidance.** Categorical. But expect NO_ORDER_ON_FILE to be dominated by orders that exist on paper and are
simply not in the queried system - so establish your order-capture completeness rate first and
report it alongside. ORDERING_PROVIDER_DECEASED and ORDERING_NPI_NOT_ENROLLED are low-volume
and high-signal; work those first regardless of dollar value. Tune to your population; a
starting point, not a standard.

**5 · Paired control + failure mode.** CD-08 (Valid-order verification, including source-verified ordering-provider identity). Secondary: CD-10 (Continued-need / recertification documentation), CD-03 (Reasonable-and-necessary documentation).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No pre-service order verification. CD-08's signed-order-plus-verified-identity artifact does not exist. |
| **Designed-not-operating** | CD-08 is per-service; a population of paid services in the period has no corresponding verification. |
| **Operated-but-bypassed** | Services proceeded on a verbal order with the written order never obtained. |
| **Operated-but-ineffective** | The check confirms an ordering NPI is present and syntactically valid but never source-verifies it against PECOS/NPPES - so stolen and deceased-provider NPIs pass. |
| **Evidence-missing** | Orders exist clinically but are not retrievable in the timeframe CD-11 requires. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 14. Agency-level hours-per-beneficiary peer outlier

| | |
|---|---|
| **Paired control** | **PI-01** - Routine claims-aberrancy analytics (peer-outlier / statistical) |
| **Control type** | Detective |
| **Module** | Module 02 - Program Integrity |
| **Owner** | PI data analyst |
| **Frequency** | Monthly |
| **System touchpoint** | Claims/encounter analytics |
| **Evidence artifact** | Outlier-alert disposition log |
| **Control failure mode** | Undetected aberrant billing |
| **Regulatory anchor** | 42 CFR 455 Subpart A (Medicaid PI) [binding]; CMS PIM Pub 100-08 Ch.2 (Data Analysis) [guidance] |
| **Secondary controls** | MD-14 - Utilization control & SURS medical-review program; PI-02 - Provider risk-scoring & review-prioritization model |

**1 · Data hypothesis.** Individual-visit tests miss the agency that bills a defensible amount per visit but far more
visits per beneficiary than its peers. Where routine claims-aberrancy analytics are absent or
run without a peer-group definition, this drift is invisible. This is the family's MEVIA-style
peer-outlier test.

**2 · What to check.** Compute hours billed per beneficiary per month for each billing agency, compute the mean and
standard deviation within a peer group (service type × region × program), and report agencies
above a z-score cutoff.

**3 · Reference SQL.**

```sql
-- TEST 14 - Peer-group outlier: hours per beneficiary per month, by agency
-- Params: :period_start, :period_end, :minutes_per_unit (15), :z_threshold (2.5),
--         :min_beneficiaries (20), :min_peer_group_size (10)
-- Portability: DATE_TRUNC is PostgreSQL/Snowflake/DuckDB style; elsewhere use DATETRUNC (SQL Server),
--   TRUNC(date, 'MM') (Oracle) or DATE_TRUNC(date, MONTH) (BigQuery).
-- Portability: JOIN ... USING (...) is not accepted by SQL Server; write the ON equalities out there.

WITH agency_month AS (
  SELECT
      cl.billing_npi,
      pr.peer_region,
      cl.payer_program,
      DATE_TRUNC('month', cl.service_date) AS service_month,
      COUNT(DISTINCT cl.beneficiary_id)    AS beneficiaries,
      SUM(COALESCE(cl.service_minutes, cl.units * :minutes_per_unit)) / 60.0 AS billed_hours,
      SUM(cl.paid_amount) AS paid_amount
  FROM claim_lines cl
  JOIN provider pr ON pr.npi = cl.billing_npi
  WHERE cl.service_date BETWEEN :period_start AND :period_end
    AND cl.service_category IN ('personal_care','attendant_care','home_health_aide','hcbs_in_home')
    AND cl.adjudication_status = 'PAID'
  GROUP BY cl.billing_npi, pr.peer_region, cl.payer_program,
           DATE_TRUNC('month', cl.service_date)
  HAVING COUNT(DISTINCT cl.beneficiary_id) >= :min_beneficiaries
),
rates AS (
  SELECT *, billed_hours / NULLIF(beneficiaries, 0) AS hours_per_beneficiary
  FROM agency_month
),
peer AS (
  SELECT peer_region, payer_program, service_month,
         AVG(hours_per_beneficiary)    AS peer_mean,
         STDDEV(hours_per_beneficiary) AS peer_sd,
         COUNT(*)                      AS peer_n
  FROM rates
  GROUP BY peer_region, payer_program, service_month
  HAVING COUNT(*) >= :min_peer_group_size          -- a peer group of 3 is not a peer group
)
SELECT
    r.billing_npi, r.peer_region, r.payer_program, r.service_month,
    r.beneficiaries,
    ROUND(r.hours_per_beneficiary, 2) AS hours_per_beneficiary,
    ROUND(p.peer_mean, 2)             AS peer_mean,
    ROUND((r.hours_per_beneficiary - p.peer_mean) / NULLIF(p.peer_sd, 0), 2) AS z_score,
    p.peer_n,
    r.paid_amount
FROM rates r
JOIN peer p USING (peer_region, payer_program, service_month)
WHERE (r.hours_per_beneficiary - p.peer_mean) / NULLIF(p.peer_sd, 0) > :z_threshold
ORDER BY z_score DESC;
```

**4 · Threshold guidance.** :z_threshold = 2.5 is the conventional starting cut. :min_beneficiaries = 20 and
:min_peer_group_size = 10 matter more than the z-cut: a z-score computed over a handful of
agencies is noise dressed as statistics. Because the hours-per-beneficiary distribution is
usually right-skewed, consider a robust alternative (median + MAD, or a raw percentile cut at
the 95th) before defending a z-score. Persistence across ≥3 months is a far stronger signal
than a single-month spike. Tune to your population; a starting point, not a standard.

**5 · Paired control + failure mode.** PI-01 (Routine claims-aberrancy analytics - peer-outlier / statistical). Secondary: MD-14 (Utilization control & SURS), PI-02 (Provider risk-scoring & review-prioritization model).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No routine peer-outlier analytics. PI-01's outlier-alert disposition log does not exist. |
| **Designed-not-operating** | PI-01 is specified monthly; no run occurred in the period. |
| **Operated-but-bypassed** | Outlier alerts were generated and closed without review to manage volume. |
| **Operated-but-ineffective** | The peer group is defined too broadly (all providers statewide) so genuine outliers regress to the mean and never clear the cut. |
| **Evidence-missing** | Alerts exist in a BI tool with no retained, dated disposition record. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 15. New-agency billing velocity in the first 90 days

| | |
|---|---|
| **Paired control** | **PE-15** - New-provider billing-velocity watch (first 60-90 days) |
| **Control type** | Detective |
| **Module** | Module 01 - Provider Enrollment |
| **Owner** | PI data analyst |
| **Frequency** | Continuous |
| **System touchpoint** | Claims analytics |
| **Evidence artifact** | Velocity-alert disposition |
| **Control failure mode** | Rapid bust-out billing post-enrollment |
| **Regulatory anchor** | DOJ 2025 Operation Gold Rush [case evidence]; Program Integrity Alliance - Value-for-Money in Anti-Fraud Efforts [case evidence] |
| **Secondary controls** | PE-16 - Acquisition-velocity flag (owner acquiring many numbers fast); PI-02 - Provider risk-scoring & review-prioritization model |

**1 · Data hypothesis.** Fly-by-night home-care agencies bill hard and fast before any review cycle reaches them. Where
the new-provider billing-velocity watch is absent, an agency can reach peak billing within
weeks of enrollment and exit before the first revalidation. The signature is a billing ramp far
steeper than the peer norm in the first 60–90 days.

**2 · What to check.** For agencies enrolled within the lookback, compute cumulative paid amount and distinct
beneficiaries by days-since-enrollment, and compare the 90-day figure to the peer-cohort
distribution for agencies of the same type and region.

**3 · Reference SQL.**

```sql
-- TEST 15 - New-agency billing velocity (first 90 days)
-- Params: :period_start, :period_end, :velocity_window_days (90), :percentile_cut (0.95)
-- Portability: date +/- integer days and date - date (a day count) are PostgreSQL/DuckDB style;
--   elsewhere use DATEADD / DATE_ADD for the offset and DATEDIFF / DATE_DIFF for the difference.

WITH new_agencies AS (
  SELECT npi AS billing_npi, enrollment_effective_date, peer_region, provider_type
  FROM provider
  WHERE enrollment_effective_date BETWEEN :period_start AND :period_end
    AND provider_type IN ('HOME_HEALTH_AGENCY','PERSONAL_CARE_AGENCY','HCBS_PROVIDER')
),
early_billing AS (
  SELECT
      n.billing_npi, n.enrollment_effective_date, n.peer_region, n.provider_type,
      SUM(cl.paid_amount)               AS paid_first_window,
      COUNT(DISTINCT cl.beneficiary_id) AS beneficiaries_first_window,
      COUNT(DISTINCT cl.rendering_npi)  AS aides_first_window,
      MIN(cl.service_date)              AS first_service_date,
      (MIN(cl.service_date) - n.enrollment_effective_date) AS days_to_first_claim
  FROM new_agencies n
  JOIN claim_lines cl
    ON  cl.billing_npi   = n.billing_npi
    AND cl.service_date >= n.enrollment_effective_date
    AND cl.service_date <  n.enrollment_effective_date + :velocity_window_days
  WHERE cl.adjudication_status = 'PAID'
  GROUP BY n.billing_npi, n.enrollment_effective_date, n.peer_region, n.provider_type
)
SELECT
    e.*,
    PERCENT_RANK() OVER (PARTITION BY provider_type, peer_region
                         ORDER BY paid_first_window) AS cohort_percentile
FROM early_billing e
QUALIFY PERCENT_RANK() OVER (PARTITION BY provider_type, peer_region
                             ORDER BY paid_first_window) >= :percentile_cut
ORDER BY paid_first_window DESC;

-- QUALIFY is Snowflake/BigQuery/Teradata. In Postgres/SQL Server, wrap the
-- SELECT in a subquery and filter on cohort_percentile in the outer WHERE.
```

**4 · Threshold guidance.** :percentile_cut = 0.95 within a same-type, same-region cohort is more defensible than any
absolute dollar figure, because it self-calibrates to your market. days_to_first_claim below ~7
is an independent signal worth surfacing separately - an agency billing within a week of
enrollment had a patient panel before it was enrolled. A high beneficiaries-to-aides ratio in
the first window is a third. Tune to your population; a starting point, not a standard.

**5 · Paired control + failure mode.** PE-15 (New-provider billing-velocity watch, first 60-90 days). Secondary: PE-16 (Acquisition-velocity flag), PI-02 (Provider risk-scoring & review-prioritization model).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No velocity watch on newly enrolled providers. |
| **Designed-not-operating** | PE-15 exists on paper; no watch list was produced for the cohort enrolled in the period. |
| **Operated-but-bypassed** | Velocity flags raised no prepayment review or site visit. |
| **Operated-but-ineffective** | The watch uses a fixed statewide dollar threshold rather than a cohort percentile, so it misses fast-ramping agencies in low-cost regions. |
| **Evidence-missing** | No retained watch list or disposition. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 16. Recertification lapse on continuing in-home services

| | |
|---|---|
| **Paired control** | **CD-10** - Continued-need / recertification documentation control for ongoing services (home health, DME rental, hospice) |
| **Control type** | Detective |
| **Module** | Module 05 - Coverage & Documentation |
| **Owner** | Clinician / supplier |
| **Frequency** | Per interval |
| **System touchpoint** | EHR |
| **Evidence artifact** | Continued-need / recert note |
| **Control failure mode** | Ongoing services billed without current documented need |
| **Regulatory anchor** | Medicare Benefit Policy Manual (CMS Pub 100-02) [guidance] |
| **Secondary controls** | CD-03 - Reasonable-and-necessary documentation control: the medical record …; MD-14 - Utilization control & SURS medical-review program |

**1 · Data hypothesis.** Ongoing in-home services require periodic re-documentation of continued need. Where the
continued-need control is absent or tracked in a system disconnected from claims, services
continue to be billed past the recertification interval with no current documented need - the
largest quiet driver of insufficient-documentation improper payments in this family.

**2 · What to check.** For each beneficiary with continuous service, compute days since the most recent
recertification/reassessment as at each service date, and flag services billed beyond the
required interval. Report the paid amount in the lapsed window, which is the exposure figure.

**3 · Reference SQL.**

```sql
-- TEST 16 - Services billed past the recertification interval
-- Params: :period_start, :period_end, :recert_interval_days (varies by program - set per HCPCS group)
-- Portability: date +/- integer days and date - date (a day count) are PostgreSQL/DuckDB style;
--   elsewhere use DATEADD / DATE_ADD for the offset and DATEDIFF / DATE_DIFF for the difference.

WITH latest_recert AS (
  SELECT
      r.beneficiary_id,
      r.hcpcs_group,
      r.recert_date,
      LEAD(r.recert_date) OVER (PARTITION BY r.beneficiary_id, r.hcpcs_group
                                ORDER BY r.recert_date) AS next_recert_date
  FROM recertification r
)
SELECT
    cl.beneficiary_id, cl.billing_npi, cl.hcpcs, cl.service_date, cl.paid_amount,
    lr.recert_date         AS last_recert_date,
    (cl.service_date - lr.recert_date) AS days_since_recert,
    :recert_interval_days  AS required_interval_days,
    (cl.service_date - lr.recert_date) - :recert_interval_days AS days_lapsed
FROM claim_lines cl
LEFT JOIN latest_recert lr
       ON  lr.beneficiary_id = cl.beneficiary_id
       AND lr.hcpcs_group    = cl.hcpcs_group
       AND cl.service_date >= lr.recert_date
       AND (lr.next_recert_date IS NULL OR cl.service_date < lr.next_recert_date)
WHERE cl.service_date BETWEEN :period_start AND :period_end
  AND cl.service_category IN ('personal_care','attendant_care','home_health_aide',
                              'private_duty_nursing','hcbs_in_home')
  AND cl.adjudication_status = 'PAID'
  AND (lr.recert_date IS NULL
       OR (cl.service_date - lr.recert_date) > :recert_interval_days)
ORDER BY days_lapsed DESC NULLS FIRST, cl.paid_amount DESC;

-- Roll up by beneficiary and by agency: exposure = SUM(paid_amount) in the lapsed window.
```

**4 · Threshold guidance.** :recert_interval_days is NOT a tunable judgement call - it is set by program rule and differs
by service and by state waiver (home-health certification periods, HCBS annual person-centred
plan reassessment, private-duty nursing). Set it per hcpcs_group from your own coverage policy;
do not use a single global value. A grace period of 15–30 days before flagging is reasonable to
absorb scheduling reality, but the grace period is an operational tolerance, not a coverage
rule. Tune to your population; a starting point, not a standard.

**5 · Paired control + failure mode.** CD-10 (Continued-need / recertification documentation control for ongoing services - home health, DME rental, hospice). Secondary: CD-03 (Reasonable-and-necessary documentation), MD-14 (Utilization control & SURS).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No recertification tracking exists for in-home services. |
| **Designed-not-operating** | CD-10 is per-interval; recertifications were not performed for a population of active beneficiaries. |
| **Operated-but-bypassed** | Recertification was completed as a form exercise with no reassessment of need documented. |
| **Operated-but-ineffective** | Tracking lives in the care-management system and never blocks or flags claims, so a lapse has no payment consequence. |
| **Evidence-missing** | Reassessments occurred but the continued-need note is not retained or retrievable. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## Gap notes - controls this family needs that the library does not yet have

*Each note names a control objective these tests point to that the control library does not yet state. They are candidates for a future library release; the library itself is unchanged by this pack.*

### G1. No dedicated EVV-reconciliation control in the control library.

The Diagnosis Matrix names EVV reconciliation as the control fix for the impossible-day,
phantom-billing and hospice/home-health flag families, and records that no control in the
library carries it (nearest: PI-24, CD-09). The nearest library controls (CB-08, CD-09, MD-24,
PI-24) each cover a slice, but no control states the EVV-to-claims reconciliation objective
directly, and EVV is a federal Medicaid mandate (21st Century Cures Act §12006 → SSA §1903(l)),
not an optional practice.

**Recommendation.** A candidate control for the library: a dedicated EVV visit-verification and billed-vs-verified
reconciliation control in Module 05 (Coverage & Documentation) or Module 07 (Medicaid & CHIP),
anchored to Cures Act §12006 / SSA §1903(l), with the impossible-day and interval-overlap rules
named in its design. TESTS 1, 2, 3, 4 and 12 in this pack are paired to the nearest existing
control until such a control exists.

### G2. Module 05 / Module 07 hold no home-health-specific time-integrity control.

The control library has strong DMEPOS, coding, enrollment and managed-care coverage, but home-
health/attendant-care time integrity - the single largest Medicaid FWA case volume in the
project's own case base - is served only indirectly, through generic quantity edits (CB-08),
generic proof-of-service (CD-09) and one self-directed-care conflict control (MD-24).

**Recommendation.** A candidate set of four controls: EVV reconciliation; interval-overlap monitoring; attendant-
roster exclusion screening (TEST 7's companion pass has no owning control at all); and service-
plan-to-delivery utilization matching. The case-base evidence for all four is already
assembled.

---

*Reference patterns. Public-source, personal-capacity, advisory and organizational only. Queries must be adapted to your own schema and tested on your own warehouse before use. Outputs are review candidates for human adjudication, never payment or denial dispositions. Confirm every regulatory anchor against its primary source.*