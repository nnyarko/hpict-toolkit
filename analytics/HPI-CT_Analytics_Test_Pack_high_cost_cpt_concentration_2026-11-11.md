# Analytics Test Pack - High-cost CPT concentration - spinal fusion, cardiac EP, and high-cost diagnostics

*where a small number of codes carry a large share of the dollars*

*Generated 2026-11-11 from the HPI-CT control library (`data/controls.csv`, 173 controls). 16 detection tests, paired to 16 primary controls. Reference patterns, not programs.*

## Before you run anything

**These queries produce review candidates, never payment or denial decisions.** Every hit is a question for a human.

**The toolkit never receives your data.** These are patterns you run inside your own environment. No claim data, no PHI, no identifiers leave your systems.

**Every query is a pattern, not a program.** Field names are generic; yours will differ. Expect to rewrite the FROM and JOIN clauses and to remap service categories to your own taxonomy.

**Every threshold is a starting point, not a standard.** Where a threshold is arithmetic it is labelled as such. Everywhere else, measure your own distribution first.

Read `PREFACE_how_to_run_these.md` before the first run.

## Scope of this family

Procedures and diagnostics with high per-unit reimbursement and material clinical discretion:
spinal fusion and instrumentation, cardiac electrophysiology (ablation and implantable
devices), molecular pathology and genetic testing, and other high-reimbursement CPT clusters.
The unit of concern here is not the aberrant claim but the aberrant *concentration* - a
provider or facility whose case mix, level count, or referral pattern diverges from peers on
codes where a single case moves more money than a hundred office visits.

This family carries the worst signal-to-abrasion ratio in the whole pack, and it is important
to say so before any of it runs. High-cost procedures are performed by specialists whose case
mix legitimately varies with referral patterns, tertiary-centre status and sub-specialisation.
Volume alone proves nothing. Every test here is built to produce a *review queue for clinical
adjudication*, and several are explicitly designed to be run as governance findings against the
payer's own controls rather than as provider findings. Where a test can only be resolved by a
clinician reading a record, the threshold guidance says so.

## Tests at a glance

| # | Test | Primary control | Also touches |
|---|---|---|---|
| 1 | Single-code revenue concentration - one CPT carrying an outsized share of a provider's paid dollars | **CB-21** | PI-01, PI-02 |
| 2 | Volume ramp - step change in a high-cost code after a policy or coverage change | **CB-17** | CD-02, CB-21 |
| 3 | Spinal fusion - levels and instrumentation add-ons per case above the peer norm | **CB-07** | CB-16, CB-08 |
| 4 | Spinal fusion with no documented conservative care in the lookback | **CD-03** | CD-05, CD-16 |
| 5 | Cardiac EP ablation without a supporting arrhythmia diagnosis or prior drug trial | **CB-11** | CD-01, CD-16 |
| 6 | Implantable cardiac device inside an NCD waiting period | **CD-01** | CD-02, PI-04 |
| 7 | Molecular pathology and genetic-test tier concentration | **CD-16** | PI-04, CB-21 |
| 8 | High-cost diagnostic with no treating relationship - ordering-NPI fan-out | **CD-08** | PI-20, CD-16 |
| 9 | Prepayment review coverage rate on high-cost codes | **PI-04** | CB-17, CD-16 |
| 10 | Post-payment review blind spot - high-cost codes never sampled | **PI-05** | PI-10, CB-19 |
| 11 | Facility-professional claim pairing mismatch on high-cost procedures | **CD-17** | CB-15, CB-13 |
| 12 | Site-of-service migration toward the higher-paying setting | **CB-13** | CB-21, CB-17 |
| 13 | Referral concentration into an entity with ownership or payment ties | **PI-23** | PI-22, PE-05 |
| 14 | Severity capture drift on high-cost admissions (CC/MCC rate vs peers) | **CB-14** | CB-19, PI-01 |
| 15 | Composite high-cost outlier score feeding the review queue | **PI-02** | PI-01, CB-21 |
| 16 | Device credit and warranty replacement not reflected on the claim | **PI-07** | CB-17, PI-09 |

> **Control-ID validation:** every control referenced above resolves to a real ID in the current control library. See the gap notes at the end for controls this family *needs* but the library does not yet have.

---

## 1. Single-code revenue concentration - one CPT carrying an outsized share of a provider's paid dollars

| | |
|---|---|
| **Paired control** | **CB-21** - High-risk-code monitoring dashboard to compliance/governance |
| **Control type** | Detective |
| **Module** | Module 03 - Coding & Billing Integrity |
| **Owner** | PI data analyst |
| **Frequency** | Monthly |
| **System touchpoint** | BI / analytics dashboard |
| **Evidence artifact** | Governance monitoring pack |
| **Control failure mode** | Emerging coding risk unseen by leadership |
| **Regulatory anchor** | OIG A-09-22-03010 (21 Jun 2023) - molecular pathology (CPT 81408); all three recommendations closed [case evidence]; GAO Green Book (2025) - Standards for Internal Control [guidance] |
| **Secondary controls** | PI-01 - Routine claims-aberrancy analytics (peer-outlier / statistical); PI-02 - Provider risk-scoring & review-prioritization model |

**1 · Data hypothesis.** Where high-risk-code monitoring is absent or reports volume rather than concentration, a
provider whose practice has narrowed onto a single high-reimbursement code is invisible to
leadership. The signature is not high volume - a busy spine surgeon has high volume - but a
high *share* of one provider's total paid dollars flowing through one or two codes, against
peers with a broader distribution.

**2 · What to check.** For each rendering provider, compute paid dollars by CPT as a share of that provider's total
paid dollars, restrict to codes on the high-cost register, and benchmark the concentration
share within specialty. Report the Herfindahl-style concentration index alongside the top-code
share so a two-code practice is not mistaken for a diversified one.

**3 · Reference SQL.**

```sql
-- TEST 1 - Single-code revenue concentration vs specialty peers
-- Params: :period_start, :period_end, :min_paid (100000), :min_peer_n (15), :z_threshold (2.5)
-- Portability: JOIN ... USING (...) is not accepted by SQL Server; write the ON equalities out there.

WITH provider_code AS (
  SELECT
      cl.rendering_npi, pr.specialty, pr.peer_region,
      cl.cpt,
      SUM(cl.paid_amount) AS code_paid,
      COUNT(*)            AS code_lines
  FROM claim_lines cl
  JOIN provider pr ON pr.npi = cl.rendering_npi
  WHERE cl.service_date BETWEEN :period_start AND :period_end
    AND cl.adjudication_status = 'PAID'
  GROUP BY cl.rendering_npi, pr.specialty, pr.peer_region, cl.cpt
),
code_share AS (
  -- share computed one step earlier: a window function cannot sit inside an aggregate
  SELECT pc.*,
         pc.code_paid / NULLIF(SUM(pc.code_paid) OVER (PARTITION BY pc.rendering_npi), 0)
           AS code_share
  FROM provider_code pc
),
provider_total AS (
  SELECT rendering_npi, specialty, peer_region,
         SUM(code_paid) AS total_paid,
         -- Herfindahl index of the provider's own code mix: 1.0 = a single-code practice
         SUM(POWER(code_share, 2)) AS code_concentration_index
  FROM code_share
  GROUP BY rendering_npi, specialty, peer_region
  HAVING SUM(code_paid) >= :min_paid
),
top_code AS (
  SELECT rendering_npi, cpt AS top_cpt, code_paid, code_lines,
         ROW_NUMBER() OVER (PARTITION BY rendering_npi ORDER BY code_paid DESC) AS rk
  FROM provider_code
),
shares AS (
  SELECT
      t.rendering_npi, pt.specialty, pt.peer_region,
      tc.top_cpt, tc.code_lines AS top_code_cases,
      tc.code_paid / NULLIF(pt.total_paid, 0) AS top_code_share,
      pt.total_paid, pt.code_concentration_index,
      hc.hcpcs IS NOT NULL AS top_code_is_high_cost
  FROM provider_total pt
  JOIN top_code tc  ON tc.rendering_npi = pt.rendering_npi AND tc.rk = 1
  JOIN provider_total t ON t.rendering_npi = pt.rendering_npi
  LEFT JOIN high_cost_code_register hc ON hc.hcpcs = tc.top_cpt
),
peer AS (
  SELECT specialty, peer_region,
         AVG(top_code_share) AS peer_mean, STDDEV(top_code_share) AS peer_sd, COUNT(*) AS peer_n
  FROM shares GROUP BY specialty, peer_region HAVING COUNT(*) >= :min_peer_n
)
SELECT
    s.rendering_npi, s.specialty, s.peer_region,
    s.top_cpt, s.top_code_cases,
    ROUND(s.top_code_share, 3)           AS top_code_share,
    ROUND(k.peer_mean, 3)                AS peer_mean_share,
    ROUND((s.top_code_share - k.peer_mean) / NULLIF(k.peer_sd, 0), 2) AS z_score,
    ROUND(s.code_concentration_index, 3) AS concentration_index,
    s.total_paid, s.top_code_is_high_cost
FROM shares s
JOIN peer k USING (specialty, peer_region)
WHERE s.top_code_is_high_cost
  AND (s.top_code_share - k.peer_mean) / NULLIF(k.peer_sd, 0) > :z_threshold
ORDER BY z_score DESC, s.total_paid DESC;
```

**4 · Threshold guidance.** :z_threshold = 2.5 within a same-specialty, same-region peer group, with :min_paid = 100,000
and :min_peer_n = 15. Read the concentration_index alongside the z-score: a sub-specialist who
does one thing well will show a high top-code share and a legitimately high concentration
index, and that is a description of their practice, not a finding. What makes a hit interesting
is concentration *plus* something else - a recent shift in concentration (test 2), an unusual
level count (test 3), or a coverage-criteria gap (tests 4–6). Treat this test as a population
filter that feeds the others, not as a standalone output. Tune to your population; a starting
point, not a standard.

**5 · Paired control + failure mode.** CB-21 (High-risk-code monitoring dashboard to compliance/governance). Secondary: PI-01 (Routine claims-aberrancy analytics), PI-02 (Provider risk-scoring & review-prioritization model).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No high-risk-code monitoring exists. CB-21's governance monitoring pack does not exist, so leadership has no view of where the high-cost dollars concentrate. |
| **Designed-not-operating** | CB-21 is specified monthly; no monitoring pack was produced for the period. |
| **Operated-but-bypassed** | The pack reached governance and no review, edit change or education followed any flagged concentration. |
| **Operated-but-ineffective** | The dashboard reports volume and total spend rather than concentration against peers, so a narrowed practice never stands out. |
| **Evidence-missing** | The dashboard exists live but no dated pack and no record of what governance saw or decided is retained. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 2. Volume ramp - step change in a high-cost code after a policy or coverage change

| | |
|---|---|
| **Paired control** | **CB-17** - Prepayment edit maintenance & tuning from emerging vulnerabilities |
| **Control type** | Preventive |
| **Module** | Module 03 - Coding & Billing Integrity |
| **Owner** | Payer PI / edit governance |
| **Frequency** | Quarterly |
| **System touchpoint** | Prepayment edit library |
| **Evidence artifact** | Edit-change board record |
| **Control failure mode** | Known vulnerabilities left unedited |
| **Regulatory anchor** | CMS Pub 100-08 Program Integrity Manual [guidance]; OIG A-09-21-03021 (2 May 2023) - psychotherapy/telehealth; all six recommendations closed [case evidence] |
| **Secondary controls** | CD-02 - Coverage-policy change monitoring: track the Medicare Coverage …; CB-21 - High-risk-code monitoring dashboard to compliance/governance |

**1 · Data hypothesis.** Where prepayment edit maintenance does not track emerging vulnerabilities, a coverage
expansion, a new code, a competitor's exit or a removed prior-auth requirement produces a step
change in volume that nobody edits against. The signature is a discontinuity, not a level: a
provider or market whose monthly volume of a high-cost code jumps and stays jumped.

**2 · What to check.** For each high-cost code, compute monthly paid volume per provider and per market, compare each
month to a trailing baseline, and flag sustained step changes - requiring the elevation to
persist for several months so a single busy month does not register.

**3 · Reference SQL.**

```sql
-- TEST 2 - Sustained step change in high-cost code volume
-- Params: :period_start, :period_end, :baseline_months (6), :ramp_multiple (2.0),
--         :sustain_months (3), :min_baseline_cases (10)
-- Portability: DATE_TRUNC is PostgreSQL/Snowflake/DuckDB style; elsewhere use DATETRUNC (SQL Server),
--   TRUNC(date, 'MM') (Oracle) or DATE_TRUNC(date, MONTH) (BigQuery).
-- The run length is set once, by :sustain_months; the window frame derives its bound from it
-- as (:sustain_months - 1) FOLLOWING, so there is no second value to keep in step.
-- Portability: a bind parameter or expression as a window frame bound is accepted by PostgreSQL
-- and DuckDB; where the bound must be a literal, write the numbers in (6 PRECEDING, 2 FOLLOWING).

WITH monthly AS (
  SELECT
      cl.rendering_npi,
      cl.cpt,
      DATE_TRUNC('month', cl.service_date) AS service_month,
      COUNT(*)            AS cases,
      SUM(cl.paid_amount) AS paid_amount
  FROM claim_lines cl
  JOIN high_cost_code_register hc ON hc.hcpcs = cl.cpt
  WHERE cl.service_date BETWEEN :period_start AND :period_end
    AND cl.adjudication_status = 'PAID'
  GROUP BY cl.rendering_npi, cl.cpt, DATE_TRUNC('month', cl.service_date)
),
with_baseline AS (
  SELECT
      m.*,
      AVG(cases) OVER (
        PARTITION BY rendering_npi, cpt ORDER BY service_month
        ROWS BETWEEN :baseline_months PRECEDING AND 1 PRECEDING
      ) AS trailing_baseline_cases,
      COUNT(*) OVER (
        PARTITION BY rendering_npi, cpt ORDER BY service_month
        ROWS BETWEEN :baseline_months PRECEDING AND 1 PRECEDING
      ) AS baseline_months_available,
      LAG(cases) OVER (PARTITION BY rendering_npi, cpt ORDER BY service_month) AS prev_month_cases,
      -- the lowest volume over this month and the next (:sustain_months - 1) months
      MIN(cases) OVER (
        PARTITION BY rendering_npi, cpt ORDER BY service_month
        ROWS BETWEEN CURRENT ROW AND (:sustain_months - 1) FOLLOWING
      ) AS min_cases_in_run,
      COUNT(*) OVER (
        PARTITION BY rendering_npi, cpt ORDER BY service_month
        ROWS BETWEEN CURRENT ROW AND (:sustain_months - 1) FOLLOWING
      ) AS months_in_run
  FROM monthly m
),
run_starts AS (
  -- a month where every month of a :sustain_months run is at least :ramp_multiple times
  -- the baseline measured BEFORE the run began; the baseline is then frozen at that point
  SELECT *,
         ROW_NUMBER() OVER (PARTITION BY rendering_npi, cpt ORDER BY service_month) AS rk
  FROM with_baseline
  WHERE baseline_months_available >= :baseline_months
    AND trailing_baseline_cases >= :min_baseline_cases
    AND months_in_run >= :sustain_months
    AND prev_month_cases < trailing_baseline_cases * :ramp_multiple   -- the run starts here, not already under way
    AND min_cases_in_run >= trailing_baseline_cases * :ramp_multiple
),
first_run AS (
  SELECT rendering_npi, cpt, service_month AS ramp_started, trailing_baseline_cases
  FROM run_starts WHERE rk = 1
)
SELECT
    f.rendering_npi, f.cpt,
    f.ramp_started,
    COUNT(*)                                   AS ramp_months,
    ROUND(f.trailing_baseline_cases, 1)        AS baseline_cases_per_month,
    ROUND(AVG(m.cases), 1)                     AS ramp_cases_per_month,
    SUM(m.paid_amount)                         AS paid_in_ramp_months
FROM first_run f
JOIN monthly m ON m.rendering_npi = f.rendering_npi AND m.cpt = f.cpt
              AND m.service_month >= f.ramp_started
              AND m.cases >= f.trailing_baseline_cases * :ramp_multiple
GROUP BY f.rendering_npi, f.cpt, f.ramp_started, f.trailing_baseline_cases
HAVING COUNT(*) >= :sustain_months
ORDER BY paid_in_ramp_months DESC;

-- Run the SAME query aggregated by market (peer_region) instead of by provider FIRST.
-- A ramp visible across an entire market is a coverage/policy event and belongs to
-- CB-17 and CD-02 - not to any provider's review queue.
```

**4 · Threshold guidance.** :ramp_multiple = 2.0 against a 6-month trailing baseline, sustained for :sustain_months = 3,
with :min_baseline_cases = 10 so that a provider going from one case to three does not register
as a doubling. :sustain_months is the only run-length setting: the query derives its window
bound from it. Growth is not misconduct: a practice that hires a second surgeon doubles
legitimately. The value of this test is in the market-level pass, which tells you whether you
are looking at a provider or at a policy change you have not edited for. Run that first,
always. Tune to your population; a starting point, not a standard.

**5 · Paired control + failure mode.** CB-17 (Prepayment edit maintenance & tuning from emerging vulnerabilities). Secondary: CD-02 (Coverage-policy change monitoring), CB-21 (High-risk-code monitoring dashboard).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No mechanism converts an emerging volume pattern into an edit. CB-17's edit-change board record does not exist. |
| **Designed-not-operating** | CB-17 is quarterly; no board convened or recorded a decision during the ramp. |
| **Operated-but-bypassed** | The ramp was observed and the edit decision was deferred with no documented risk acceptance. |
| **Operated-but-ineffective** | Edit review is triggered only by external findings (OIG/GAO) and never by the payer's own volume signals, so a novel local pattern is structurally out of scope. |
| **Evidence-missing** | Edits changed during the period but no record connects the change to the vulnerability that prompted it. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 3. Spinal fusion - levels and instrumentation add-ons per case above the peer norm

| | |
|---|---|
| **Paired control** | **CB-07** - MUE units-of-service edit enforcement |
| **Control type** | Preventive |
| **Module** | Module 03 - Coding & Billing Integrity |
| **Owner** | Billing system owner |
| **Frequency** | Per claim |
| **System touchpoint** | Claim scrubber / edit engine |
| **Evidence artifact** | MUE hit / adjudication log |
| **Control failure mode** | Payment for medically-unlikely quantities |
| **Regulatory anchor** | CMS NCCI Medically Unlikely Edits (MUE) [guidance] |
| **Secondary controls** | CB-16 - Unbundling detection analytics - incl. same-sample …; CB-08 - Quantity / frequency-limit edits (capped & time-based services) |

**1 · Data hypothesis.** Fusion payment scales with the number of levels and the instrumentation billed. Where MUE and
add-on-code enforcement are absent or applied per line, a surgeon can bill additional-level
add-on codes and interbody-device codes at counts above what the operative case supports. The
signature is a per-case level count distribution shifted right of peers - visible only when
add-on codes are aggregated back to the case, which most edit engines do not do.

**2 · What to check.** Reconstruct the surgical case (beneficiary + date + surgeon), sum the primary and add-on level
codes and the interbody-device codes, and compare the per-case level count to the specialty
peer distribution.

**3 · Reference SQL.**

```sql
-- TEST 3 - Spinal fusion levels per case vs peers
-- Code sets are illustrative: primary fusion (22551, 22554, 22558, 22612, 22630, 22633),
-- additional-level add-ons (22552, 22585, 22614, 22632, 22634), interbody device
-- (22853, 22854, 22859), instrumentation (22840-22847). VERIFY against current CPT.
-- Params: :period_start, :period_end, :min_cases (10), :min_peer_n (15), :z_threshold (2.5)
-- Portability: JOIN ... USING (...) is not accepted by SQL Server; write the ON equalities out there.

WITH fusion_case AS (
  SELECT
      cl.beneficiary_id, cl.rendering_npi, cl.service_date,
      SUM(CASE WHEN cl.cpt IN ('22551','22554','22558','22612','22630','22633')
               THEN cl.units ELSE 0 END) AS primary_levels,
      SUM(CASE WHEN cl.cpt IN ('22552','22585','22614','22632','22634')
               THEN cl.units ELSE 0 END) AS additional_levels,
      SUM(CASE WHEN cl.cpt IN ('22853','22854','22859')
               THEN cl.units ELSE 0 END) AS interbody_devices,
      SUM(CASE WHEN cl.cpt BETWEEN '22840' AND '22847'
               THEN cl.units ELSE 0 END) AS instrumentation_units,
      SUM(cl.paid_amount) AS case_paid
  FROM claim_lines cl
  WHERE cl.service_date BETWEEN :period_start AND :period_end
    AND cl.adjudication_status = 'PAID'
    AND cl.cpt IN ('22551','22552','22554','22558','22585','22612','22614','22630',
                   '22632','22633','22634','22853','22854','22859',
                   '22840','22841','22842','22843','22844','22845','22846','22847')
  GROUP BY cl.beneficiary_id, cl.rendering_npi, cl.service_date
  HAVING SUM(CASE WHEN cl.cpt IN ('22551','22554','22558','22612','22630','22633')
                  THEN cl.units ELSE 0 END) > 0        -- a primary fusion anchors the case
),
by_surgeon AS (
  SELECT
      f.rendering_npi, pr.specialty, pr.peer_region,
      COUNT(*)                                      AS cases,
      AVG(f.primary_levels + f.additional_levels)   AS mean_levels_per_case,
      AVG(f.interbody_devices)                      AS mean_devices_per_case,
      MAX(f.primary_levels + f.additional_levels)   AS max_levels_in_a_case,
      SUM(f.case_paid)                              AS paid_amount
  FROM fusion_case f
  JOIN provider pr ON pr.npi = f.rendering_npi
  GROUP BY f.rendering_npi, pr.specialty, pr.peer_region
  HAVING COUNT(*) >= :min_cases
),
peer AS (
  SELECT specialty, peer_region,
         AVG(mean_levels_per_case) AS peer_mean, STDDEV(mean_levels_per_case) AS peer_sd,
         COUNT(*) AS peer_n
  FROM by_surgeon GROUP BY specialty, peer_region HAVING COUNT(*) >= :min_peer_n
)
SELECT
    s.rendering_npi, s.specialty, s.peer_region, s.cases,
    ROUND(s.mean_levels_per_case, 2)  AS mean_levels_per_case,
    ROUND(k.peer_mean, 2)             AS peer_mean_levels,
    ROUND((s.mean_levels_per_case - k.peer_mean) / NULLIF(k.peer_sd, 0), 2) AS z_score,
    ROUND(s.mean_devices_per_case, 2) AS mean_devices_per_case,
    s.max_levels_in_a_case, s.paid_amount
FROM by_surgeon s
JOIN peer k USING (specialty, peer_region)
WHERE (s.mean_levels_per_case - k.peer_mean) / NULLIF(k.peer_sd, 0) > :z_threshold
ORDER BY z_score DESC;
```

**4 · Threshold guidance.** :z_threshold = 2.5 with :min_cases = 10. The CPT lists above are illustrative and MUST be
verified against the current code set before use - fusion coding is revised regularly and an
out-of-date add-on list silently under-counts levels. Deformity and revision surgery
legitimately carry high level counts, so exclude or stratify scoliosis and revision-fusion
cases before benchmarking, or the output will be a list of deformity surgeons.
`max_levels_in_a_case` is worth eyeballing independently: a single implausibly high case is an
anatomy question with a definite answer. Tune to your population; a starting point, not a
standard.

**5 · Paired control + failure mode.** CB-07 (MUE units-of-service edit enforcement). Secondary: CB-16 (Unbundling detection analytics), CB-08 (Quantity / frequency-limit edits).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No units enforcement on add-on and device codes. CB-07's MUE hit/adjudication log does not exist for this code family. |
| **Designed-not-operating** | MUE values are loaded but the quarterly refresh did not run for the period. |
| **Operated-but-bypassed** | Unit overrides were granted at adjudication without operative-report review. |
| **Operated-but-ineffective** | Edits apply per claim line and never aggregate add-on codes back to the surgical case, so level stacking across lines passes untouched. This is the defining defect for this test. |
| **Evidence-missing** | No retained adjudication log showing which unit hits fired and how they were resolved. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 4. Spinal fusion with no documented conservative care in the lookback

| | |
|---|---|
| **Paired control** | **CD-03** - Reasonable-and-necessary documentation control: the medical record substantiates the R&N basis at the point of service |
| **Control type** | Preventive |
| **Module** | Module 05 - Coverage & Documentation |
| **Owner** | Treating clinician |
| **Frequency** | Per service |
| **System touchpoint** | EHR |
| **Evidence artifact** | R&N-supporting progress note |
| **Control failure mode** | Insufficient-documentation improper payment (SSA 1862(a)(1)(A)) |
| **Regulatory anchor** | SSA 1862(a)(1)(A) [binding]; Medicare Program Integrity Manual (CMS Pub 100-08) Ch.3 [guidance] |
| **Secondary controls** | CD-05 - Item/service-category documentation-eligibility checklist to prevent …; CD-16 - Enhanced documentation review for high-cost / high-risk tests and … |

**1 · Data hypothesis.** Most coverage policies for elective lumbar fusion require a documented trial of conservative
management - physical therapy, injections, medication - over a defined period before surgery.
Where the reasonable-and-necessary documentation control is absent or is satisfied by a
checkbox, fusions proceed with no conservative-care footprint anywhere in the beneficiary's
claim history.

**2 · What to check.** For each elective fusion case, look back over the policy-defined window for claims evidencing
conservative management, and flag cases with none. Exclude trauma, tumour, infection and
neurological-deficit indications, where conservative care is not expected.

**3 · Reference SQL.**

```sql
-- TEST 4 - Elective fusion with no conservative-care claims in the lookback
-- This test infers documentation from CLAIMS, which is a proxy. A clean claim history
-- does not prove the record is absent - care may have been delivered elsewhere or
-- self-paid. Output is a chart-review queue, nothing more.
-- Params: :period_start, :period_end, :conservative_lookback_days (180)
-- Portability: date +/- integer days and date - date (a day count) are PostgreSQL/DuckDB style;
--   elsewhere use DATEADD / DATE_ADD for the offset and DATEDIFF / DATE_DIFF for the difference.

SELECT
    f.claim_line_id, f.beneficiary_id, f.rendering_npi, f.cpt AS fusion_cpt,
    f.service_date, f.paid_amount,
    dx.icd10_code AS primary_icd10,
    (SELECT COUNT(*) FROM claim_lines c
      WHERE c.beneficiary_id = f.beneficiary_id
        AND c.service_date BETWEEN f.service_date - :conservative_lookback_days
                               AND f.service_date - 1
        AND c.adjudication_status = 'PAID'
        AND c.cpt IN (SELECT hcpcs FROM conservative_care_code_list)) AS conservative_claims
FROM claim_lines f
JOIN claim_diagnosis dx ON dx.claim_id = f.claim_id AND dx.diagnosis_sequence = 1
WHERE f.service_date BETWEEN :period_start AND :period_end
  AND f.adjudication_status = 'PAID'
  AND f.cpt IN ('22551','22554','22558','22612','22630','22633')       -- primary fusion
  -- exclude indications where conservative care is not expected
  AND dx.icd10_code NOT IN (SELECT icd10_code FROM fusion_urgent_indication_list)
  AND NOT EXISTS (
    SELECT 1 FROM claim_lines c
    WHERE c.beneficiary_id = f.beneficiary_id
      AND c.service_date BETWEEN f.service_date - :conservative_lookback_days
                             AND f.service_date - 1
      AND c.adjudication_status = 'PAID'
      AND c.cpt IN (SELECT hcpcs FROM conservative_care_code_list)
  )
ORDER BY f.paid_amount DESC;

-- conservative_care_code_list: PT/OT (97110, 97112, 97140, 97530), epidural and
-- facet injections (62321-62323, 64483-64484, 64490-64495), chiropractic (98940-98942),
-- plus your own additions. Build it with clinical input, not from this comment.
```

**4 · Threshold guidance.** :conservative_lookback_days = 180 is a common policy window but is NOT universal - read it off
the applicable NCD/LCD or plan policy rather than adopting the default. Two structural caveats
govern how far this test can be pushed. First, absence of a claim is not absence of care:
therapy delivered under a different payer, self-paid, or bundled into a global episode leaves
no trace here. Second, the exclusion lists do the real work - build
`conservative_care_code_list` and `fusion_urgent_indication_list` with clinical input before
the first run. This is a chart-review queue, and it should never be described as anything else.
Tune to your population; a starting point, not a standard.

**5 · Paired control + failure mode.** CD-03 (Reasonable-and-necessary documentation control). Secondary: CD-05 (Item/service-category documentation-eligibility checklist), CD-16 (Enhanced documentation review for high-cost / high-risk services).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No R&N documentation control operates for elective high-cost surgery. CD-03's R&N-supporting note requirement is not enforced for this category. |
| **Designed-not-operating** | CD-03 is per-service; a population of paid fusion cases has no corresponding documentation verification. |
| **Operated-but-bypassed** | Conservative-care attestation was accepted as a form field with no underlying record. |
| **Operated-but-ineffective** | The control requires documentation to exist but never tests whether it substantiates the specific coverage criterion - presence, not sufficiency. |
| **Evidence-missing** | Documentation exists clinically but is not retrievable within the CD-11 retention and retrieval requirement. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 5. Cardiac EP ablation without a supporting arrhythmia diagnosis or prior drug trial

| | |
|---|---|
| **Paired control** | **CB-11** - Diagnosis-to-service medical-necessity linkage edit (NCD/LCD) |
| **Control type** | Preventive |
| **Module** | Module 03 - Coding & Billing Integrity |
| **Owner** | Coding / edit builder |
| **Frequency** | Per claim |
| **System touchpoint** | Claim scrubber vs NCD/LCD |
| **Evidence artifact** | Necessity-edit disposition |
| **Control failure mode** | Payment without supporting diagnosis |
| **Regulatory anchor** | SSA 1862(a)(1)(A) (reasonable & necessary) [binding] |
| **Secondary controls** | CD-01 - Coverage-criteria application: identify & apply the current …; CD-16 - Enhanced documentation review for high-cost / high-risk tests and … |

**1 · Data hypothesis.** Catheter ablation is covered for defined arrhythmias, and many policies expect a documented
trial of, or contraindication to, antiarrhythmic therapy. Where the diagnosis-to-service
linkage edit is absent or checks only the primary diagnosis position, ablations pay with no
supporting arrhythmia diagnosis anywhere on the claim and no pharmacological history.

**2 · What to check.** For each ablation claim, test whether any diagnosis position carries a covered arrhythmia code,
and separately whether the beneficiary has any antiarrhythmic drug claim in the lookback.
Report the two conditions independently - they fail for different reasons and point at
different controls.

**3 · Reference SQL.**

```sql
-- TEST 5 - Ablation without supporting diagnosis or prior antiarrhythmic therapy
-- Ablation CPTs (illustrative): 93653, 93654, 93656 (comprehensive EP with ablation),
-- 93655, 93657 (add-on). VERIFY against current CPT and the applicable LCD.
-- Params: :period_start, :period_end, :drug_lookback_days (365)
-- Portability: date +/- integer days and date - date (a day count) are PostgreSQL/DuckDB style;
--   elsewhere use DATEADD / DATE_ADD for the offset and DATEDIFF / DATE_DIFF for the difference.

SELECT
    a.claim_line_id, a.beneficiary_id, a.rendering_npi, a.billing_npi,
    a.cpt AS ablation_cpt, a.service_date, a.paid_amount,
    CASE WHEN EXISTS (
        SELECT 1 FROM claim_diagnosis d
        JOIN covered_arrhythmia_dx cd ON cd.icd10_code = d.icd10_code
        WHERE d.claim_id = a.claim_id
    ) THEN 1 ELSE 0 END AS has_covered_arrhythmia_dx,
    CASE WHEN EXISTS (
        SELECT 1 FROM pharmacy_claim rx
        JOIN antiarrhythmic_drug_list ad ON ad.ndc = rx.ndc
        WHERE rx.beneficiary_id = a.beneficiary_id
          AND rx.fill_date BETWEEN a.service_date - :drug_lookback_days AND a.service_date
    ) THEN 1 ELSE 0 END AS has_prior_antiarrhythmic,
    CASE
      WHEN NOT EXISTS (SELECT 1 FROM claim_diagnosis d
                       JOIN covered_arrhythmia_dx cd ON cd.icd10_code = d.icd10_code
                       WHERE d.claim_id = a.claim_id)
        THEN 'NO_SUPPORTING_DIAGNOSIS'      -- edit failure: CB-11
      ELSE 'NO_PRIOR_DRUG_TRIAL'            -- coverage-criteria question: CD-01, chart review
    END AS finding_class
FROM claim_lines a
WHERE a.service_date BETWEEN :period_start AND :period_end
  AND a.adjudication_status = 'PAID'
  AND a.cpt IN ('93653','93654','93655','93656','93657')
  AND (
    NOT EXISTS (SELECT 1 FROM claim_diagnosis d
                JOIN covered_arrhythmia_dx cd ON cd.icd10_code = d.icd10_code
                WHERE d.claim_id = a.claim_id)
    OR NOT EXISTS (SELECT 1 FROM pharmacy_claim rx
                   JOIN antiarrhythmic_drug_list ad ON ad.ndc = rx.ndc
                   WHERE rx.beneficiary_id = a.beneficiary_id
                     AND rx.fill_date BETWEEN a.service_date - :drug_lookback_days
                                          AND a.service_date)
  )
ORDER BY finding_class, a.paid_amount DESC;
```

**4 · Threshold guidance.** NO_SUPPORTING_DIAGNOSIS is close to categorical and is an edit-configuration finding - the
claim should not have adjudicated. NO_PRIOR_DRUG_TRIAL is not a finding at all on its own: drug
intolerance, contraindication, and patient preference are all legitimate paths to first-line
ablation, and pharmacy data may sit with a different payer entirely. Work the diagnosis bucket
first; treat the drug bucket as a chart-review sample sized to your review capacity, not as a
worklist. If you do not hold pharmacy claims, drop that half of the test rather than
substituting a weaker proxy. Tune to your population; a starting point, not a standard.

**5 · Paired control + failure mode.** CB-11 (Diagnosis-to-service medical-necessity linkage edit - NCD/LCD). Secondary: CD-01 (Coverage-criteria application), CD-16 (Enhanced documentation review for high-cost / high-risk services).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No necessity-linkage edit covers this procedure family. CB-11's necessity-edit disposition does not exist. |
| **Designed-not-operating** | The edit is specified but not active in the adjudication path for these codes. |
| **Operated-but-bypassed** | Edit hits were overridden without a documented coverage rationale. |
| **Operated-but-ineffective** | The edit inspects the primary diagnosis position only, so a supporting arrhythmia code in a secondary position fails the edit while an unsupported claim with the right code in position one passes. |
| **Evidence-missing** | No retained record of edit hits and dispositions for this code family. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 6. Implantable cardiac device inside an NCD waiting period

| | |
|---|---|
| **Paired control** | **CD-01** - Coverage-criteria application: identify & apply the current NCD/LCD/LCA and Benefit Policy Manual criteria for each item/service before billing |
| **Control type** | Preventive |
| **Module** | Module 05 - Coverage & Documentation |
| **Owner** | Coding / Medical reviewer |
| **Frequency** | Per service |
| **System touchpoint** | Coverage-rules engine + MCD |
| **Evidence artifact** | Coverage-criteria crosswalk |
| **Control failure mode** | Billing a service that is not covered / not reasonable-and-necessary |
| **Regulatory anchor** | SSA 1862(a)(1)(A) [binding]; Medicare Coverage Database (NCD/LCD/LCA) [guidance] |
| **Secondary controls** | CD-02 - Coverage-policy change monitoring: track the Medicare Coverage …; PI-04 - Prepayment medical-review edits on high-risk codes/services |

**1 · Data hypothesis.** The ICD national coverage determination imposes waiting periods - for example after myocardial
infarction, after revascularisation, and after a new diagnosis of heart failure - before a
prophylactic implant is covered. Where the coverage-criteria control is absent or applies stale
criteria, implants pay inside those windows. This is the single most litigated coverage-
criteria area in cardiology and the basis of a national hospital settlement programme.

**2 · What to check.** For each device implant, look back for a qualifying index event - MI, CABG, PCI, or first
heart-failure diagnosis - and flag implants falling inside the applicable waiting period,
excluding the indications that override it.

**3 · Reference SQL.**

```sql
-- TEST 6 - ICD implant inside an NCD waiting period
-- Waiting periods and exceptions are set by the applicable NCD/LCD and change over time.
-- Load them as a versioned reference table (ncd_waiting_period); do not hardcode.
-- Event dating: MI, CABG and PCI are dated at the LATEST qualifying claim before the implant.
-- A first-diagnosis event (NEW_HF_DX) is dated at the EARLIEST qualifying claim held, so a
-- long-standing heart-failure patient with a recent pre-operative visit is not read as new.
-- That is only as good as the claims history you hold: require enough look-back per patient.
-- Params: :period_start, :period_end
-- Portability: date +/- integer days and date - date (a day count) are PostgreSQL/DuckDB style;
--   elsewhere use DATEADD / DATE_ADD for the offset and DATEDIFF / DATE_DIFF for the difference.

WITH implants AS (
  SELECT cl.claim_line_id, cl.claim_id, cl.beneficiary_id, cl.rendering_npi,
         cl.billing_npi, cl.cpt, cl.service_date, cl.paid_amount
  FROM claim_lines cl
  WHERE cl.service_date BETWEEN :period_start AND :period_end
    AND cl.adjudication_status = 'PAID'
    AND cl.cpt IN ('33249','33262','33263','33264','33270','33271')   -- verify against current CPT
),
event_claims AS (
  SELECT DISTINCT
      c.beneficiary_id, c.claim_id,
      e.event_type,                       -- 'MI' | 'CABG' | 'PCI' | 'NEW_HF_DX'
      c.service_date AS event_date
  FROM claim_lines c
  JOIN index_event_definition e
    ON (e.match_on = 'CPT'   AND e.code = c.cpt)
    OR (e.match_on = 'ICD10' AND e.code IN (SELECT icd10_code FROM claim_diagnosis d
                                            WHERE d.claim_id = c.claim_id))
  WHERE c.adjudication_status = 'PAID'
),
index_events AS (
  -- per implant and event type: one event date taken from claims dated BEFORE the implant day
  -- (the implant claim and other same-day claims often carry the indication code; later
  -- claims are ignored). First-diagnosis event types take the first claim, the rest the latest.
  SELECT i.claim_line_id, ev.event_type,
         CASE WHEN ev.event_type IN ('NEW_HF_DX')        -- list every first-occurrence event type here
              THEN MIN(ev.event_date)
              ELSE MAX(ev.event_date) END AS event_date
  FROM implants i
  JOIN event_claims ev ON ev.beneficiary_id = i.beneficiary_id
                      AND ev.event_date <  i.service_date
                      AND ev.claim_id  <> i.claim_id
  GROUP BY i.claim_line_id, ev.event_type
)
SELECT
    i.claim_line_id, i.beneficiary_id, i.rendering_npi, i.billing_npi,
    i.cpt AS implant_cpt, i.service_date AS implant_date, i.paid_amount,
    ie.event_type, ie.event_date,
    (i.service_date - ie.event_date) AS days_since_event,
    w.waiting_period_days,
    w.policy_id
FROM implants i
JOIN index_events ie ON ie.claim_line_id = i.claim_line_id
JOIN ncd_waiting_period w
  ON  w.event_type = ie.event_type
  AND i.service_date BETWEEN w.effective_date
                         AND COALESCE(w.retirement_date, DATE '9999-12-31')
WHERE (i.service_date - ie.event_date) < w.waiting_period_days
  -- exclusions that override the waiting period (e.g. secondary prevention after
  -- documented sustained VT/VF, or an existing device requiring replacement)
  AND NOT EXISTS (
    SELECT 1 FROM claim_diagnosis d
    JOIN waiting_period_exception_dx x ON x.icd10_code = d.icd10_code
    WHERE d.claim_id = i.claim_id
  )
ORDER BY days_since_event ASC, i.paid_amount DESC;
```

**4 · Threshold guidance.** Categorical against the loaded policy, and unusually consequential - which is exactly why the
exception handling has to be right before anyone acts. Secondary prevention indications,
generator replacements, and device upgrades all sit outside the waiting-period logic and will
otherwise dominate the output. Confirm the waiting-period values against the current NCD for
the service date rather than the current version; they have changed. Restrict the first run to
primary prophylactic implants with a clean index event, and expand only once the exclusion
lists are validated clinically. NEW_HF_DX is dated at the earliest heart-failure claim the
warehouse holds, so a patient whose history starts recently (new to the plan) can look newly
diagnosed: set a minimum period of prior enrolment before trusting that event type. An index
event dated on the implant day itself is not counted. Tune to your population; a starting
point, not a standard.

**5 · Paired control + failure mode.** CD-01 (Coverage-criteria application - identify and apply the current NCD/LCD/LCA and Benefit Policy Manual criteria). Secondary: CD-02 (Coverage-policy change monitoring), PI-04 (Prepayment medical-review edits on high-risk codes/services).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No coverage-criteria crosswalk exists for this service. CD-01's crosswalk artifact does not exist. |
| **Designed-not-operating** | CD-01 is per-service; a population of paid implants has no coverage-criteria determination on file. |
| **Operated-but-bypassed** | Criteria were applied and a non-conforming case proceeded on clinical override with no documented basis. |
| **Operated-but-ineffective** | The crosswalk encodes a superseded version of the NCD, so the control runs correctly against the wrong rule - CD-02 is the upstream failure. |
| **Evidence-missing** | Determinations were made but no retained crosswalk record shows which criteria were applied to which case. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 7. Molecular pathology and genetic-test tier concentration

| | |
|---|---|
| **Paired control** | **CD-16** - Enhanced documentation review for high-cost / high-risk tests and services |
| **Control type** | Detective |
| **Module** | Module 05 - Coverage & Documentation |
| **Owner** | PI / Medical reviewer |
| **Frequency** | Per claim/sample |
| **System touchpoint** | Pre-payment edit |
| **Evidence artifact** | Enhanced-review workpaper |
| **Control failure mode** | High-dollar improper payment for unsupported high-cost service (OIG molecular pathology) |
| **Regulatory anchor** | OIG A-09-22-03010 (21 Jun 2023) - molecular pathology genetic test; all three recommendations closed [case evidence] |
| **Secondary controls** | PI-04 - Prepayment medical-review edits on high-risk codes/services; CB-21 - High-risk-code monitoring dashboard to compliance/governance |

**1 · Data hypothesis.** Molecular pathology codes are tiered by complexity, with the top tiers paying many multiples of
the base. Where enhanced documentation review for high-cost tests is absent, a laboratory can
bill the highest tier as a default. CD-16 and PI-04 both cite the OIG molecular-pathology
finding (CPT 81408) directly, so the vulnerability is already named in the control library  - 
only the query is missing.

**2 · What to check.** For each billing laboratory, compute the share of molecular pathology claims billed at the top
tier and the paid-per-test average, benchmark within the laboratory peer group, and flag
concentration. Report the beneficiary-level panel count as a companion signal - many high-tier
tests for one beneficiary in a short window is a distinct pattern.

**3 · Reference SQL.**

```sql
-- TEST 7 - Top-tier molecular pathology concentration by laboratory
-- Tier 1 molecular pathology: 81200-81383; Tier 2: 81400-81408 (81408 = highest tier,
-- the OIG-named code). Genomic sequencing procedures: 81410-81471. VERIFY current CPT.
-- Params: :period_start, :period_end, :min_tests (50), :top_tier_share_cut (0.50)

WITH mp AS (
  SELECT
      cl.billing_npi, cl.ordering_npi, cl.beneficiary_id, cl.cpt,
      cl.service_date, cl.paid_amount,
      CASE WHEN cl.cpt IN ('81407','81408') THEN 1.0 ELSE 0 END AS is_top_tier
  FROM claim_lines cl
  WHERE cl.service_date BETWEEN :period_start AND :period_end
    AND cl.adjudication_status = 'PAID'
    AND (cl.cpt BETWEEN '81200' AND '81408' OR cl.cpt BETWEEN '81410' AND '81471')
),
by_lab AS (
  SELECT
      billing_npi,
      COUNT(*)                          AS tests,
      AVG(is_top_tier)                  AS top_tier_share,
      SUM(paid_amount)                  AS paid_amount,
      SUM(paid_amount) / NULLIF(COUNT(*), 0) AS paid_per_test,
      COUNT(DISTINCT beneficiary_id)    AS beneficiaries,
      COUNT(DISTINCT ordering_npi)      AS ordering_providers
  FROM mp
  GROUP BY billing_npi
  HAVING COUNT(*) >= :min_tests
)
SELECT
    billing_npi, tests, beneficiaries, ordering_providers,
    ROUND(top_tier_share, 3) AS top_tier_share,
    ROUND(paid_per_test, 2)  AS paid_per_test,
    ROUND(tests * 1.0 / NULLIF(beneficiaries, 0), 2) AS tests_per_beneficiary,
    paid_amount
FROM by_lab
WHERE top_tier_share > :top_tier_share_cut
ORDER BY paid_amount DESC;

-- Companion: beneficiaries receiving more than :panel_threshold distinct high-tier
-- molecular tests within :panel_window_days. Panel-stacking on a single beneficiary is a
-- different pattern from a lab's overall tier mix, and points at the ordering provider
-- rather than the lab.
```

**4 · Threshold guidance.** :top_tier_share_cut = 0.50 and :min_tests = 50 are starting cuts. Specialisation is the
confound: a reference laboratory that exists to run rare-disease panels will legitimately show
a high top-tier share, and the right comparison is against other reference laboratories, not
against hospital labs. `tests_per_beneficiary` and the `ordering_providers` count are the more
discriminating columns - a lab with a high tier share, few ordering providers and many tests
per beneficiary is a materially different picture from one with the same tier share spread
across a wide referral base. Tune to your population; a starting point, not a standard.

**5 · Paired control + failure mode.** CD-16 (Enhanced documentation review for high-cost / high-risk tests and services). Secondary: PI-04 (Prepayment medical-review edits on high-risk codes/services), CB-21 (High-risk-code monitoring dashboard).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No enhanced review operates for high-cost diagnostics. CD-16's enhanced-review workpaper does not exist. |
| **Designed-not-operating** | CD-16 is per-claim/sample; no sample was reviewed for this category in the period. |
| **Operated-but-bypassed** | Reviews were performed and no denial, edit change or referral followed a sustained pattern. |
| **Operated-but-ineffective** | Review is triggered by a per-claim dollar threshold that the individual test does not cross, so a high-volume, moderate-value pattern is never selected. |
| **Evidence-missing** | Reviews occurred but no workpaper records which claims were selected or what was concluded. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 8. High-cost diagnostic with no treating relationship - ordering-NPI fan-out

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
| **Secondary controls** | PI-20 - Provider NPI/credential-footprint consistency monitoring - flag …; CD-16 - Enhanced documentation review for high-cost / high-risk tests and … |

**1 · Data hypothesis.** Where valid-order verification is absent, or verifies only that an ordering NPI field is
populated, a small number of ordering providers can generate a very large volume of high-cost
diagnostics for beneficiaries they have no treating relationship with. The signature is an
ordering NPI with many distinct beneficiaries and few or no corresponding evaluation
encounters.

**2 · What to check.** For each ordering provider on high-cost diagnostic claims, count distinct beneficiaries ordered
for, count how many of those beneficiaries had any evaluation encounter with that provider in
the lookback, and report the no-encounter ratio and the distinct-laboratory fan-out.

**3 · Reference SQL.**

```sql
-- TEST 8 - Ordering-provider fan-out on high-cost diagnostics
-- Deliberately scoped to DIAGNOSTICS. The DMEPOS/telemedicine order-to-encounter test is
-- family 6; keep them separate so the paired controls stay distinct.
-- Params: :period_start, :period_end, :encounter_lookback_days (180), :min_orders (25),
--         :no_encounter_cut (0.80)
-- Portability: date +/- integer days and date - date (a day count) are PostgreSQL/DuckDB style;
--   elsewhere use DATEADD / DATE_ADD for the offset and DATEDIFF / DATE_DIFF for the difference.

WITH orders AS (
  SELECT
      cl.ordering_npi, cl.beneficiary_id, cl.billing_npi AS lab_npi,
      cl.claim_line_id, cl.service_date, cl.paid_amount,
      CASE WHEN EXISTS (
        SELECT 1 FROM claim_lines e
        WHERE e.beneficiary_id = cl.beneficiary_id
          AND e.rendering_npi  = cl.ordering_npi
          AND e.claim_type IN ('PROFESSIONAL','TELEHEALTH')
          AND e.cpt BETWEEN '99202' AND '99215'
          AND e.adjudication_status = 'PAID'
          AND e.service_date BETWEEN cl.service_date - :encounter_lookback_days
                                 AND cl.service_date
      ) THEN 1 ELSE 0 END AS has_treating_encounter
  FROM claim_lines cl
  JOIN high_cost_code_register hc ON hc.hcpcs = cl.cpt AND hc.category = 'DIAGNOSTIC'
  WHERE cl.service_date BETWEEN :period_start AND :period_end
    AND cl.adjudication_status = 'PAID'
    AND cl.ordering_npi IS NOT NULL
)
SELECT
    o.ordering_npi, pr.specialty, pr.peer_region,
    COUNT(*)                                   AS orders,
    COUNT(DISTINCT o.beneficiary_id)           AS distinct_beneficiaries,
    COUNT(DISTINCT o.lab_npi)                  AS distinct_labs,
    ROUND(1 - AVG(o.has_treating_encounter), 3) AS no_encounter_ratio,
    SUM(o.paid_amount)                         AS paid_amount
FROM orders o
LEFT JOIN provider pr ON pr.npi = o.ordering_npi
GROUP BY o.ordering_npi, pr.specialty, pr.peer_region
HAVING COUNT(*) >= :min_orders
   AND (1 - AVG(o.has_treating_encounter)) > :no_encounter_cut
ORDER BY paid_amount DESC;
```

**4 · Threshold guidance.** :no_encounter_cut = 0.80 and :min_orders = 25 are starting cuts. The :encounter_lookback_days
window is the sensitive parameter: 180 days is generous for a diagnostic order, but a shorter
window will flag legitimate follow-up ordering and a longer one will miss the pattern entirely.
Ordering providers with high fan-out across many distinct laboratories are the strongest
candidates - a single-lab relationship usually has a mundane explanation. Note that
pathologists, radiologists and hospitalists order legitimately without an E&M encounter;
exclude those specialties or the output is a list of them. Tune to your population; a starting
point, not a standard.

**5 · Paired control + failure mode.** CD-08 (Valid-order verification - signed order on file, source-verified ordering-provider identity). Secondary: PI-20 (Provider NPI/credential-footprint consistency monitoring), CD-16 (Enhanced documentation review for high-cost tests).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No order verification operates for diagnostics. CD-08's signed-order-plus-verified-identity artifact does not exist. |
| **Designed-not-operating** | CD-08 is per-service; a population of paid high-cost diagnostics has no verification record. |
| **Operated-but-bypassed** | Tests were performed on a requisition with no signed order subsequently obtained. |
| **Operated-but-ineffective** | The check confirms the ordering NPI is present and valid but never source-verifies it against PECOS/NPPES nor tests for a treating relationship - so stolen and rented NPIs pass. |
| **Evidence-missing** | Orders exist but are not retrievable within the retention requirement. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 9. Prepayment review coverage rate on high-cost codes

| | |
|---|---|
| **Paired control** | **PI-04** - Prepayment medical-review edits on high-risk codes/services |
| **Control type** | Preventive |
| **Module** | Module 02 - Program Integrity |
| **Owner** | Medical-review / SIU |
| **Frequency** | Continuous |
| **System touchpoint** | Claims adjudication edits |
| **Evidence artifact** | Edit-hit & denial report |
| **Control failure mode** | Improper payment before any recovery |
| **Regulatory anchor** | OIG A-09-22-03010 (21 Jun 2023) - molecular pathology genetic test (CPT 81408); all three recommendations closed [case evidence]; OIG A-09-21-03006 (27 Feb 2023) - definitive drug testing (G0483); all four recommendations closed [case evidence] |
| **Secondary controls** | CB-17 - Prepayment edit maintenance & tuning from emerging vulnerabilities; CD-16 - Enhanced documentation review for high-cost / high-risk tests and … |

**1 · Data hypothesis.** This test measures the control, not the claims. Where prepayment medical-review edits are
absent or configured too narrowly, high-cost codes adjudicate straight through with no review
touch at all. The finding is a coverage rate: what share of high-cost paid dollars passed
through any prepayment review, and which codes have a rate of effectively zero.

**2 · What to check.** For each high-cost register code, compute paid dollars, the count of claims that hit any
prepayment review edit, and the resulting denial rate. Flag codes with material spend and a
review-touch rate near zero.

**3 · Reference SQL.**

```sql
-- TEST 9 - Prepayment review coverage on high-cost codes
-- Requires the edit-hit / medical-review log. If you cannot produce this query at all,
-- that inability IS the PI-04 finding - record it as such.
-- Params: :period_start, :period_end, :min_paid (250000), :min_review_rate (0.01)

WITH spend AS (
  SELECT
      hc.hcpcs, hc.category, hc.risk_source,
      COUNT(cl.claim_line_id) AS paid_lines,
      SUM(cl.paid_amount)     AS paid_amount,
      COUNT(DISTINCT cl.billing_npi) AS billing_entities
  FROM high_cost_code_register hc
  LEFT JOIN claim_lines cl
    ON  cl.cpt = hc.hcpcs
    AND cl.service_date BETWEEN :period_start AND :period_end
    AND cl.adjudication_status = 'PAID'
  GROUP BY hc.hcpcs, hc.category, hc.risk_source
),
reviewed AS (
  SELECT
      mr.hcpcs,
      COUNT(*)                                          AS reviewed_claims,
      SUM(CASE WHEN mr.outcome = 'DENIED' THEN 1 ELSE 0 END) AS denied_claims
  FROM medical_review_log mr
  WHERE mr.review_type = 'PREPAYMENT'
    AND mr.review_date BETWEEN :period_start AND :period_end
  GROUP BY mr.hcpcs
)
SELECT
    s.hcpcs, s.category, s.risk_source,
    s.paid_lines, s.paid_amount, s.billing_entities,
    COALESCE(r.reviewed_claims, 0) AS reviewed_claims,
    ROUND(COALESCE(r.reviewed_claims, 0) * 1.0 / NULLIF(s.paid_lines, 0), 4) AS review_rate,
    ROUND(COALESCE(r.denied_claims, 0)   * 1.0 / NULLIF(r.reviewed_claims, 0), 3) AS denial_rate_when_reviewed
FROM spend s
LEFT JOIN reviewed r ON r.hcpcs = s.hcpcs
WHERE s.paid_amount >= :min_paid
  AND COALESCE(r.reviewed_claims, 0) * 1.0 / NULLIF(s.paid_lines, 0) < :min_review_rate
ORDER BY s.paid_amount DESC;

-- Read denial_rate_when_reviewed alongside the review rate. A code with a LOW review
-- rate and a HIGH denial rate among those reviewed is the strongest possible argument
-- for expanding the edit: the reviews you did do are finding problems.
```

**4 · Threshold guidance.** :min_paid = 250,000 and :min_review_rate = 0.01 are placeholders for your own materiality. The
output is a control finding by construction and should go to the edit-change board (CB-17), not
into any provider review queue. The most useful single number this query produces is
`denial_rate_when_reviewed` on a low-review-rate code - a high denial rate on a small sample is
the cleanest available evidence that the review population should be larger, and it is an
argument made entirely from the payer's own operating data. Tune to your population; a starting
point, not a standard.

**5 · Paired control + failure mode.** PI-04 (Prepayment medical-review edits on high-risk codes/services). Secondary: CB-17 (Prepayment edit maintenance & tuning), CD-16 (Enhanced documentation review for high-cost tests).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No prepayment medical-review edits operate on high-cost codes. PI-04's edit-hit & denial report does not exist. |
| **Designed-not-operating** | Edits are configured but produced no hits in the period, and no one noticed. |
| **Operated-but-bypassed** | Review hits were released to payment under throughput pressure without clinical adjudication. |
| **Operated-but-ineffective** | Edits are scoped so narrowly that they touch a negligible share of the code's volume - measurable directly by this query's review_rate. |
| **Evidence-missing** | Reviews occur but no log ties a reviewed claim to a reviewer and an outcome, so the coverage rate cannot be evidenced at all. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 10. Post-payment review blind spot - high-cost codes never sampled

| | |
|---|---|
| **Paired control** | **PI-05** - Post-payment medical review w/ valid sampling & extrapolation |
| **Control type** | Detective |
| **Module** | Module 02 - Program Integrity |
| **Owner** | Medical-review nurse/MD |
| **Frequency** | Per project |
| **System touchpoint** | MR case system |
| **Evidence artifact** | Sampling plan + extrapolation memo |
| **Control failure mode** | Unsupported claims paid, weak recovery basis |
| **Regulatory anchor** | CMS PIM Pub 100-08 Ch.3 (Verifying Errors & Corrective Actions) [guidance]; CMS PIM Pub 100-08 Ch.8 (Administrative Actions / Sampling) [guidance] |
| **Secondary controls** | PI-10 - Root-cause corrective-action workflow (beyond recoupment); CB-19 - Coding audit program (statistically valid sampling) + error-rate … |

**1 · Data hypothesis.** The mirror of test 9 on the detective side. Where post-payment medical review operates as a
series of targeted projects without portfolio coverage tracking, large categories of high-cost
spend go years without ever entering a sample. The absence is invisible precisely because each
individual project looks reasonable.

**2 · What to check.** For each high-cost code, find the date it was last included in any post-payment review project,
compute elapsed time and unreviewed paid dollars since, and rank by exposure.

**3 · Reference SQL.**

```sql
-- TEST 10 - High-cost spend with no post-payment review coverage
-- Params: :period_start, :period_end, :stale_days (730)

WITH last_review AS (
  SELECT
      mr.hcpcs,
      MAX(mr.review_date) AS last_reviewed,
      MAX(mr.project_id)  AS last_project
  FROM medical_review_log mr
  WHERE mr.review_type = 'POSTPAYMENT'
  GROUP BY mr.hcpcs
),
spend AS (
  SELECT
      hc.hcpcs, hc.category, hc.risk_source,
      SUM(cl.paid_amount)     AS paid_amount,
      COUNT(cl.claim_line_id) AS paid_lines
  FROM high_cost_code_register hc
  JOIN claim_lines cl
    ON  cl.cpt = hc.hcpcs
    AND cl.service_date BETWEEN :period_start AND :period_end
    AND cl.adjudication_status = 'PAID'
  GROUP BY hc.hcpcs, hc.category, hc.risk_source
)
SELECT
    s.hcpcs, s.category, s.risk_source,
    s.paid_lines, s.paid_amount,
    lr.last_reviewed, lr.last_project,
    CASE WHEN lr.last_reviewed IS NULL THEN NULL
         ELSE CURRENT_DATE - lr.last_reviewed END AS days_since_review,
    CASE WHEN lr.last_reviewed IS NULL THEN 'NEVER_REVIEWED'
         WHEN CURRENT_DATE - lr.last_reviewed > :stale_days THEN 'STALE'
         ELSE 'CURRENT' END AS coverage_class
FROM spend s
LEFT JOIN last_review lr ON lr.hcpcs = s.hcpcs
WHERE lr.last_reviewed IS NULL
   OR CURRENT_DATE - lr.last_reviewed > :stale_days
ORDER BY s.paid_amount DESC;
```

**4 · Threshold guidance.** :stale_days = 730 (two years) is a reasonable opening definition of stale for a high-cost code,
but the right value depends on how fast your coding and coverage landscape moves - high-cost
diagnostics turn over far faster than surgical procedures, and a single global value will
misclassify one of them. The NEVER_REVIEWED bucket, sorted by paid dollars, is the output that
matters and usually needs no threshold discussion at all. This is a review-planning input for
PI-05 and a portfolio-coverage argument for governance - it is not evidence about any provider.
Tune to your population; a starting point, not a standard.

**5 · Paired control + failure mode.** PI-05 (Post-payment medical review with valid sampling & extrapolation). Secondary: PI-10 (Root-cause corrective-action workflow), CB-19 (Coding audit program with statistically valid sampling).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No post-payment medical review programme exists. PI-05's sampling plan and extrapolation memo do not exist. |
| **Designed-not-operating** | Projects are defined but none ran in the period. |
| **Operated-but-bypassed** | Reviews identified errors and no recovery or corrective action followed (PI-09, PI-10). |
| **Operated-but-ineffective** | Project selection is driven entirely by external referrals, so portfolio coverage is never assessed and whole categories of spend are structurally excluded. |
| **Evidence-missing** | Reviews occurred but no log records which codes were in scope, so coverage cannot be demonstrated. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 11. Facility-professional claim pairing mismatch on high-cost procedures

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
| **Secondary controls** | CB-15 - Duplicate claim / line detection; CB-13 - Place-of-service / facility-vs-non-facility rate edit |

**1 · Data hypothesis.** A high-cost procedure normally generates both a facility claim and a professional claim. Where
the coordination/setting control is absent, one side can appear without the other - a facility
claim with no surgeon, or a surgeon's claim with no facility - which points variously at
services not rendered, at claims submitted under the wrong entity, or at a setting
misstatement.

**2 · What to check.** For each high-cost procedure on a facility claim, look for a corresponding professional claim
for the same beneficiary and date, and vice versa. Report both directions separately; they mean
different things.

**3 · Reference SQL.**

```sql
-- TEST 11 - Facility and professional claims that should pair, and do not
-- Params: :period_start, :period_end, :pairing_window_days (1)
-- Portability: date +/- integer days and date - date (a day count) are PostgreSQL/DuckDB style;
--   elsewhere use DATEADD / DATE_ADD for the offset and DATEDIFF / DATE_DIFF for the difference.

-- 11a: facility claim with no professional claim
SELECT
    'FACILITY_WITHOUT_PROFESSIONAL' AS mismatch_type,
    f.claim_id, f.beneficiary_id, f.facility_npi AS entity_npi,
    f.cpt, f.service_date, f.paid_amount
FROM claim_lines f
JOIN high_cost_code_register hc ON hc.hcpcs = f.cpt
WHERE f.service_date BETWEEN :period_start AND :period_end
  AND f.adjudication_status = 'PAID'
  AND f.claim_type IN ('OUTPATIENT','ASC','INPATIENT')
  AND NOT EXISTS (
    SELECT 1 FROM claim_lines p
    WHERE p.beneficiary_id = f.beneficiary_id
      AND p.cpt            = f.cpt
      AND p.claim_type     = 'PROFESSIONAL'
      AND p.adjudication_status = 'PAID'
      AND p.service_date BETWEEN f.service_date - :pairing_window_days
                             AND f.service_date + :pairing_window_days
  )

UNION ALL

-- 11b: professional claim with no facility claim, where the POS asserts a facility
SELECT
    'PROFESSIONAL_WITHOUT_FACILITY' AS mismatch_type,
    p.claim_id, p.beneficiary_id, p.billing_npi AS entity_npi,
    p.cpt, p.service_date, p.paid_amount
FROM claim_lines p
JOIN high_cost_code_register hc ON hc.hcpcs = p.cpt
WHERE p.service_date BETWEEN :period_start AND :period_end
  AND p.adjudication_status = 'PAID'
  AND p.claim_type = 'PROFESSIONAL'
  AND p.place_of_service IN ('19','21','22','24')     -- hospital / ASC settings
  AND NOT EXISTS (
    SELECT 1 FROM claim_lines f
    WHERE f.beneficiary_id = p.beneficiary_id
      AND f.cpt            = p.cpt
      AND f.claim_type IN ('OUTPATIENT','ASC','INPATIENT')
      AND f.adjudication_status = 'PAID'
      AND f.service_date BETWEEN p.service_date - :pairing_window_days
                             AND p.service_date + :pairing_window_days
  )
ORDER BY paid_amount DESC;
```

**4 · Threshold guidance.** :pairing_window_days = 1 absorbs the common case where facility and professional claims carry
service dates a day apart. Before drawing any conclusion, establish your baseline pairing rate
 -  if it is not well above 90% you have a claim-linkage problem rather than a services problem,
and the entire output is noise. Carve-out arrangements, professional-component-only billing,
and capitated facility payment all break the pairing legitimately, so exclude those
arrangements explicitly rather than discovering them one review at a time. Tune to your
population; a starting point, not a standard.

**5 · Paired control + failure mode.** CD-17 (Coordination-of-benefits / setting documentation control). Secondary: CB-15 (Duplicate claim / line detection), CB-13 (Place-of-service / facility-vs-non-facility rate edit).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No cross-setting reconciliation exists. CD-17's COB documentation artifact does not exist. |
| **Designed-not-operating** | CD-17 is per-claim; no reconciliation ran for high-cost procedures in the period. |
| **Operated-but-bypassed** | Mismatches were identified and closed without establishing which claim was correct. |
| **Operated-but-ineffective** | The control checks benefit-period overlap only and never tests whether the expected companion claim exists at all. |
| **Evidence-missing** | No retained reconciliation output. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 12. Site-of-service migration toward the higher-paying setting

| | |
|---|---|
| **Paired control** | **CB-13** - Place-of-service / facility-vs-non-facility rate edit |
| **Control type** | Detective |
| **Module** | Module 03 - Coding & Billing Integrity |
| **Owner** | Billing / PI analyst |
| **Frequency** | Continuous |
| **System touchpoint** | Claims system (CWF) edits |
| **Evidence artifact** | POS-vs-rate exception report |
| **Control failure mode** | Higher non-facility rate paid in error |
| **Regulatory anchor** | CMS Pub 100-04 Claims Processing Manual [guidance]; OIG A-04-21-04084 (30 May 2023) - POS-32 non-facility rate; recs 23-A-04-080.03 and .04 open [case evidence] |
| **Secondary controls** | CB-21 - High-risk-code monitoring dashboard to compliance/governance; CB-17 - Prepayment edit maintenance & tuning from emerging vulnerabilities |

**1 · Data hypothesis.** The same procedure pays differently across inpatient, hospital outpatient, ASC and office
settings. Where the place-of-service control operates as a rate edit but nobody monitors
utilisation shift, a provider or system can migrate a procedure into the higher-paying setting
over time with every individual claim coded correctly. Each claim is right; the pattern is the
finding.

**2 · What to check.** For each high-cost procedure, compute the setting mix by provider and by market across
successive periods, and flag providers whose mix has shifted materially toward the higher-
paying setting relative to the market trend.

**3 · Reference SQL.**

```sql
-- TEST 12 - Setting-mix shift toward the higher-paying site of service
-- Params: :period_start, :period_end, :shift_threshold (0.15), :min_cases_per_quarter (5)
-- Portability: DATE_TRUNC is PostgreSQL/Snowflake/DuckDB style; elsewhere use DATETRUNC (SQL Server),
--   TRUNC(date, 'MM') (Oracle) or DATE_TRUNC(date, MONTH) (BigQuery).
-- Portability: MIN_BY / MAX_BY are DuckDB/Snowflake/Databricks functions; elsewhere use
--   ROW_NUMBER() ... = 1 or FIRST_VALUE over the same ordering.
-- Portability: JOIN ... USING (...) is not accepted by SQL Server; write the ON equalities out there.

WITH setting_mix AS (
  SELECT
      cl.rendering_npi, cl.cpt, pr.peer_region,
      DATE_TRUNC('quarter', cl.service_date) AS service_quarter,
      COUNT(*) AS cases,
      AVG(CASE WHEN cl.place_of_service IN ('21','22') THEN 1.0 ELSE 0 END) AS hospital_share,
      SUM(cl.paid_amount) AS paid_amount
  FROM claim_lines cl
  JOIN provider pr ON pr.npi = cl.rendering_npi
  JOIN high_cost_code_register hc ON hc.hcpcs = cl.cpt
  WHERE cl.service_date BETWEEN :period_start AND :period_end
    AND cl.adjudication_status = 'PAID'
  GROUP BY cl.rendering_npi, cl.cpt, pr.peer_region, DATE_TRUNC('quarter', cl.service_date)
  HAVING COUNT(*) >= :min_cases_per_quarter
),
market AS (
  SELECT cpt, peer_region, service_quarter,
         AVG(hospital_share) AS market_hospital_share
  FROM setting_mix
  GROUP BY cpt, peer_region, service_quarter
),
paired AS (
  SELECT
      s.rendering_npi, s.cpt, s.peer_region,
      MIN(s.service_quarter) AS first_quarter,
      MAX(s.service_quarter) AS last_quarter,
      MIN_BY(s.hospital_share, s.service_quarter) AS first_hospital_share,
      MAX_BY(s.hospital_share, s.service_quarter) AS last_hospital_share,
      MIN_BY(m.market_hospital_share, s.service_quarter) AS first_market_share,
      MAX_BY(m.market_hospital_share, s.service_quarter) AS last_market_share,
      SUM(s.paid_amount) AS paid_amount
  FROM setting_mix s
  JOIN market m USING (cpt, peer_region, service_quarter)
  GROUP BY s.rendering_npi, s.cpt, s.peer_region
)
SELECT
    rendering_npi, cpt, peer_region, first_quarter, last_quarter,
    ROUND(first_hospital_share, 3) AS first_hospital_share,
    ROUND(last_hospital_share, 3)  AS last_hospital_share,
    ROUND(last_hospital_share - first_hospital_share, 3) AS provider_shift,
    ROUND(last_market_share  - first_market_share, 3)    AS market_shift,
    ROUND((last_hospital_share - first_hospital_share)
        - (last_market_share  - first_market_share), 3)  AS excess_shift,
    paid_amount
FROM paired
WHERE (last_hospital_share - first_hospital_share)
    - (last_market_share  - first_market_share) > :shift_threshold
ORDER BY excess_shift DESC, paid_amount DESC;

-- MIN_BY/MAX_BY are Snowflake/BigQuery/Databricks. In Postgres use
-- (array_agg(x ORDER BY q))[1] and (array_agg(x ORDER BY q DESC))[1];
-- in SQL Server use FIRST_VALUE/LAST_VALUE windows.
```

**4 · Threshold guidance.** :shift_threshold = 0.15 excess shift - that is, 15 percentage points of migration beyond what
the provider's own market did over the same interval. Netting off the market trend is the whole
point: site-of-service migration is often driven by hospital acquisition of physician
practices, ASC capacity, or payer policy, none of which is a provider finding, and all of which
move the market together. A provider moving with their market is not a signal; a provider
moving against it is. Even then this is a contracting and payment-policy conversation before it
is a program-integrity one. Tune to your population; a starting point, not a standard.

**5 · Paired control + failure mode.** CB-13 (Place-of-service / facility-vs-non-facility rate edit). Secondary: CB-21 (High-risk-code monitoring dashboard), CB-17 (Prepayment edit maintenance & tuning). See gap note G4 - no control monitors setting-mix *shift* as distinct from POS coding accuracy.

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No monitoring of setting mix exists in any form. |
| **Designed-not-operating** | CB-13 is specified continuous; no POS-vs-rate exception reporting covers the period. |
| **Operated-but-bypassed** | The shift was observed and no contracting, payment-policy or review response followed. |
| **Operated-but-ineffective** | CB-13 validates the rate paid for the POS asserted on each claim - which is correct and insufficient, because a correctly-coded migration is invisible to it. |
| **Evidence-missing** | No retained reporting of setting mix over time. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 13. Referral concentration into an entity with ownership or payment ties

| | |
|---|---|
| **Paired control** | **PI-23** - MSO/investment-disguised kickback detection - cross-reference referral volume against ownership/Open Payments and MSO 'investment distribution' patterns to surface referral inducements structured as investment returns |
| **Control type** | Detective |
| **Module** | Module 02 - Program Integrity |
| **Owner** | SIU / PI data analyst |
| **Frequency** | Annual / on-trigger |
| **System touchpoint** | Claims + Open Payments + ownership-disclosure data |
| **Evidence artifact** | MSO-distribution vs referral-volume correlation report |
| **Control failure mode** | Kickbacks disguised as MSO investment returns, invisible in transaction-level views (one investment-return arrangement, alleged more than $61M in DOJ settlements since 2019) |
| **Regulatory anchor** | 42 USC 1320a-7b(b) (Anti-Kickback Statute) [binding]; 42 USC 1320a-7h(a)(1), (a)(2), (c) (SSA 1128G, Open Payments: manufacturers and group purchasing organizations report payments and physician ownership in them; data input only, not payer or MSO data) [binding]; 42 CFR 403 Subpart I [binding] |
| **Secondary controls** | PI-22 - Entity-network cash-loop detection - follow-the-money analytics …; PE-05 - Beneficial-ownership network/graph analysis |

**1 · Data hypothesis.** Where kickback-detection analytics are absent, financial relationships between referring
physicians and the entities performing high-cost procedures are invisible in transaction-level
views. The signature is referral concentration that correlates with an ownership stake or a
payment stream - physician-owned distributorships in spinal implants are the canonical high-
cost instance, and PI-23 is written for exactly this cross-reference.

**2 · What to check.** For each referring provider, compute the share of their high-cost referrals going to a single
entity, join to ownership-disclosure and Open Payments data, and report where high
concentration coincides with a disclosed financial relationship.

**3 · Reference SQL.**

```sql
-- TEST 13 - Referral concentration correlated with a financial relationship
-- Open Payments (42 CFR 403 Subpart I) and your own ownership-disclosure data are the
-- reference sources. This test surfaces a QUESTION about a relationship; it does not
-- characterise the relationship. Anti-kickback conclusions require legal analysis.
-- Params: :period_start, :period_end, :min_referrals (20), :concentration_cut (0.75)

WITH referrals AS (
  SELECT
      cl.ordering_npi AS referring_npi,
      cl.billing_npi  AS performing_entity_npi,
      COUNT(*)            AS referrals,
      SUM(cl.paid_amount) AS referred_paid
  FROM claim_lines cl
  JOIN high_cost_code_register hc ON hc.hcpcs = cl.cpt
  WHERE cl.service_date BETWEEN :period_start AND :period_end
    AND cl.adjudication_status = 'PAID'
    AND cl.ordering_npi IS NOT NULL
    AND cl.ordering_npi <> cl.billing_npi
  GROUP BY cl.ordering_npi, cl.billing_npi
),
totals AS (
  SELECT referring_npi, SUM(referrals) AS total_referrals, SUM(referred_paid) AS total_paid
  FROM referrals GROUP BY referring_npi
  HAVING SUM(referrals) >= :min_referrals
)
SELECT
    r.referring_npi, r.performing_entity_npi,
    r.referrals, t.total_referrals,
    ROUND(r.referrals * 1.0 / NULLIF(t.total_referrals, 0), 3) AS concentration_share,
    r.referred_paid,
    own.ownership_pct,
    op.total_open_payments_value,
    op.payment_categories
FROM referrals r
JOIN totals t ON t.referring_npi = r.referring_npi
LEFT JOIN ownership_disclosure own
       ON own.owner_npi = r.referring_npi AND own.entity_npi = r.performing_entity_npi
LEFT JOIN open_payments_summary op
       ON op.physician_npi = r.referring_npi AND op.paying_entity_npi = r.performing_entity_npi
WHERE r.referrals * 1.0 / NULLIF(t.total_referrals, 0) > :concentration_cut
  AND (own.ownership_pct IS NOT NULL OR op.total_open_payments_value IS NOT NULL)
ORDER BY r.referred_paid DESC;

-- Run WITHOUT the financial-relationship filter first to see the concentration
-- distribution. High concentration alone is extremely common and entirely lawful - 
-- it is the conjunction with a financial tie that makes a case worth a look.
```

**4 · Threshold guidance.** :concentration_cut = 0.75 and :min_referrals = 20 are starting cuts, and this is the test where
thresholds matter least and framing matters most. Referral concentration is normal: geography,
hospital affiliation, sub-specialty and network design all produce it lawfully, and so does
simple professional trust. The query is built so that concentration alone never surfaces - a
disclosed ownership stake or an Open Payments relationship must also be present. Even then the
output is a question for counsel and the SIU, not a conclusion: physician ownership is lawful
under multiple safe harbours and disclosure is evidence of compliance, not of its opposite.
Tune to your population; a starting point, not a standard.

**5 · Paired control + failure mode.** PI-23 (MSO/investment-disguised kickback detection - cross-reference referral volume against ownership and Open Payments data). Secondary: PI-22 (Entity-network cash-loop detection), PE-05 (Beneficial-ownership network/graph analysis).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No cross-reference between referral volume and financial-relationship data exists. PI-23's correlation report does not exist. |
| **Designed-not-operating** | PI-23 is annual/on-trigger; no report was produced in the period. |
| **Operated-but-bypassed** | Correlations were identified and no SIU review, referral or legal assessment followed. |
| **Operated-but-ineffective** | The analysis uses ownership-disclosure data only and never joins Open Payments, so relationships structured as consulting or royalty payments are invisible. |
| **Evidence-missing** | The analysis was run but neither the correlation output nor its disposition is retained. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 14. Severity capture drift on high-cost admissions (CC/MCC rate vs peers)

| | |
|---|---|
| **Paired control** | **CB-14** - DRG & discharge-status coding accuracy review (transfer vs discharge) |
| **Control type** | Detective |
| **Module** | Module 03 - Coding & Billing Integrity |
| **Owner** | Inpatient coding auditor |
| **Frequency** | Monthly |
| **System touchpoint** | Claims analytics / edits |
| **Evidence artifact** | Discharge-status audit sample |
| **Control failure mode** | Full DRG paid where transfer applies |
| **Regulatory anchor** | CMS Pub 100-04 Claims Processing Manual [guidance]; OIG A-09-23-03016 (2023) - post-acute-transfer DRG [case evidence] |
| **Secondary controls** | CB-19 - Coding audit program (statistically valid sampling) + error-rate …; PI-01 - Routine claims-aberrancy analytics (peer-outlier / statistical) |

**1 · Data hypothesis.** Inpatient payment scales with severity: the same base procedure pays materially more when a
complication or comorbidity is coded. Where DRG coding accuracy review is absent or samples
only discharge status, a facility's severity capture can drift upward without any individual
claim looking wrong. The signature is a CC/MCC rate above peers on the same base DRG family,
and a rate that has moved.

**2 · What to check.** For each facility and base DRG family covering high-cost procedures, compute the share of
admissions coded with a CC or MCC, benchmark within the peer group, and report both the level
and the period-over-period change.

**3 · Reference SQL.**

```sql
-- TEST 14 - CC/MCC capture rate on high-cost admissions vs peers
-- Params: :period_start, :period_end, :min_admissions (50), :min_peer_n (15), :z_threshold (2.5)
-- Portability: DATE_TRUNC is PostgreSQL/Snowflake/DuckDB style; elsewhere use DATETRUNC (SQL Server),
--   TRUNC(date, 'MM') (Oracle) or DATE_TRUNC(date, MONTH) (BigQuery).
-- Portability: JOIN ... USING (...) is not accepted by SQL Server; write the ON equalities out there.

WITH admissions AS (
  SELECT
      i.facility_npi, f.facility_type, f.peer_region,
      d.base_drg_family,
      DATE_TRUNC('year', i.discharge_date) AS discharge_year,
      COUNT(*) AS admissions,
      AVG(CASE WHEN d.severity_level IN ('CC','MCC') THEN 1.0 ELSE 0 END) AS cc_mcc_rate,
      SUM(i.paid_amount) AS paid_amount
  FROM inpatient_claim i
  JOIN drg_reference d ON d.drg_code = i.drg_code
  JOIN facility f      ON f.npi      = i.facility_npi
  WHERE i.discharge_date BETWEEN :period_start AND :period_end
    AND i.adjudication_status = 'PAID'
    AND d.base_drg_family IN (SELECT base_drg_family FROM high_cost_drg_family)
  GROUP BY i.facility_npi, f.facility_type, f.peer_region, d.base_drg_family,
           DATE_TRUNC('year', i.discharge_date)
  HAVING COUNT(*) >= :min_admissions
),
admissions_yoy AS (
  -- year-on-year change computed before the z filter, so a facility that is newly high
  -- (not high the year before) still carries its prior-year rate
  SELECT a.*,
         ROUND(a.cc_mcc_rate - LAG(a.cc_mcc_rate) OVER (
             PARTITION BY a.facility_npi, a.base_drg_family ORDER BY a.discharge_year), 3)
           AS yoy_change
  FROM admissions a
),
peer AS (
  SELECT facility_type, peer_region, base_drg_family, discharge_year,
         AVG(cc_mcc_rate) AS peer_mean, STDDEV(cc_mcc_rate) AS peer_sd, COUNT(*) AS peer_n
  FROM admissions
  GROUP BY facility_type, peer_region, base_drg_family, discharge_year
  HAVING COUNT(*) >= :min_peer_n
)
SELECT
    a.facility_npi, a.facility_type, a.peer_region, a.base_drg_family, a.discharge_year,
    a.admissions,
    ROUND(a.cc_mcc_rate, 3) AS cc_mcc_rate,
    ROUND(p.peer_mean, 3)   AS peer_mean,
    ROUND((a.cc_mcc_rate - p.peer_mean) / NULLIF(p.peer_sd, 0), 2) AS z_score,
    a.yoy_change,
    a.paid_amount
FROM admissions_yoy a
JOIN peer p USING (facility_type, peer_region, base_drg_family, discharge_year)
WHERE (a.cc_mcc_rate - p.peer_mean) / NULLIF(p.peer_sd, 0) > :z_threshold
ORDER BY z_score DESC;
```

**4 · Threshold guidance.** :z_threshold = 2.5 with :min_admissions = 50. Case mix is the confound and it is a serious one:
a tertiary referral centre treats sicker patients and will show a genuinely higher CC/MCC rate,
which is why the peer group must control for facility type, not just region. Read `yoy_change`
as the more interesting column - a facility that has always been high is probably describing
its population, while one that moved sharply is describing a change in its coding, and a coding
change has a cause worth knowing. Improved documentation is a legitimate cause. Tune to your
population; a starting point, not a standard.

**5 · Paired control + failure mode.** CB-14 (DRG & discharge-status coding accuracy review). Secondary: CB-19 (Coding audit program with statistically valid sampling), PI-01 (Routine claims-aberrancy analytics).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No DRG coding accuracy review exists. CB-14's discharge-status audit sample does not exist. |
| **Designed-not-operating** | CB-14 is monthly; no sample was drawn in the period. |
| **Operated-but-bypassed** | Severity findings were identified and no rebilling, recovery or education followed. |
| **Operated-but-ineffective** | The review covers discharge status and transfer logic only, never severity capture, so CC/MCC drift is structurally out of scope. This is the defining defect for this test. |
| **Evidence-missing** | Audit occurred but the sample and its conclusions are not retained in reviewable form. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 15. Composite high-cost outlier score feeding the review queue

| | |
|---|---|
| **Paired control** | **PI-02** - Provider risk-scoring & review-prioritization model |
| **Control type** | Detective |
| **Module** | Module 02 - Program Integrity |
| **Owner** | PI data analyst |
| **Frequency** | Monthly |
| **System touchpoint** | Risk-scoring engine |
| **Evidence artifact** | Risk-ranked review queue |
| **Control failure mode** | Mis-targeted review effort |
| **Regulatory anchor** | CMS PIM Pub 100-08 Ch.2 (Data Analysis) [guidance] |
| **Secondary controls** | PI-01 - Routine claims-aberrancy analytics (peer-outlier / statistical); CB-21 - High-risk-code monitoring dashboard to compliance/governance |

**1 · Data hypothesis.** Any single test in this family produces too many candidates and too little confidence to act on
alone. Where provider risk-scoring is absent, review effort is allocated by whoever shouts
loudest. The composite - a provider appearing in several independent tests - is far more
discriminating than any one of them, and building it is what PI-02 exists to do.

**2 · What to check.** Union the provider-level outputs of tests 1, 3, 7, 8, 12, 13 and 14 into a signal table, count
distinct signals per provider, weight by exposure, and rank. Report which signals each provider
triggered so the queue is explainable.

**3 · Reference SQL.**

```sql
-- TEST 15 - Composite outlier score across the family's provider-level tests
-- Populate analytics_signal from the other tests in this pack: one row per
-- (provider, signal_name, run_date, exposure_amount). Keep signal_name stable - 
-- the whole value of this table is that it accumulates across runs.
-- Params: :period_start, :period_end, :min_signals (2)
-- Portability: STRING_AGG is PostgreSQL/SQL Server/DuckDB style; elsewhere use LISTAGG (Oracle,
--   Snowflake, Redshift) or GROUP_CONCAT (MySQL).

WITH signals AS (
  SELECT
      s.provider_npi,
      s.signal_name,
      s.exposure_amount,
      s.run_date
  FROM analytics_signal s
  WHERE s.run_date BETWEEN :period_start AND :period_end
    AND s.family = 'high_cost_cpt_concentration'
),
scored AS (
  SELECT
      provider_npi,
      COUNT(DISTINCT signal_name)                  AS distinct_signals,
      SUM(exposure_amount)                         AS total_exposure,
      -- string_agg / listagg / array_join depending on dialect
      STRING_AGG(DISTINCT signal_name, ', ')       AS signals_triggered
  FROM signals
  GROUP BY provider_npi
)
SELECT
    sc.provider_npi, pr.specialty, pr.peer_region,
    sc.distinct_signals,
    sc.total_exposure,
    sc.signals_triggered,
    -- deliberately simple: signal count first, exposure as the tie-break.
    -- Do not weight signals differently until you have outcome data to justify it.
    RANK() OVER (ORDER BY sc.distinct_signals DESC, sc.total_exposure DESC) AS review_rank
FROM scored sc
LEFT JOIN provider pr ON pr.npi = sc.provider_npi
WHERE sc.distinct_signals >= :min_signals
ORDER BY review_rank;
```

**4 · Threshold guidance.** :min_signals = 2 is the whole threshold, and it should stay crude. The temptation is to weight
signals by perceived severity or to fit a model; resist it until you have outcome data - review
dispositions, overturn rates, recovery outcomes - to fit against, because a weighting scheme
invented in advance encodes assumptions you cannot defend and will be asked to defend. Signal
count first, exposure as tie-break, and `signals_triggered` carried through so every position
in the queue can be explained to the provider it concerns. That explainability is a control
property, not a nicety. Tune to your population; a starting point, not a standard.

**5 · Paired control + failure mode.** PI-02 (Provider risk-scoring & review-prioritization model). Secondary: PI-01 (Routine claims-aberrancy analytics), CB-21 (High-risk-code monitoring dashboard).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No risk-scoring or prioritisation exists. PI-02's risk-ranked review queue does not exist and review effort is allocated ad hoc. |
| **Designed-not-operating** | PI-02 is monthly; no scoring cycle ran. |
| **Operated-but-bypassed** | A ranked queue exists and reviews are selected outside it without recorded justification. |
| **Operated-but-ineffective** | The model runs on a single signal, so it reproduces that signal's false positives rather than filtering them - the point of a composite is defeated. |
| **Evidence-missing** | Scores are generated but neither the inputs nor the resulting queue are retained, so prioritisation decisions cannot be reconstructed. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 16. Device credit and warranty replacement not reflected on the claim

| | |
|---|---|
| **Paired control** | **PI-07** - 60-day self-identified overpayment report-&-return workflow |
| **Control type** | Preventive |
| **Module** | Module 02 - Program Integrity |
| **Owner** | Compliance / billing |
| **Frequency** | Per event |
| **System touchpoint** | Overpayment tracking system |
| **Evidence artifact** | Refund + 60-day timestamp |
| **Control failure mode** | Retained overpayment becomes FCA exposure |
| **Regulatory anchor** | 42 CFR 401.305 + SSA 1128J(d) (60-day rule) [binding] |
| **Secondary controls** | CB-17 - Prepayment edit maintenance & tuning from emerging vulnerabilities; PI-09 - Overpayment recovery tracking vs reopening / RAC look-back |

**1 · Data hypothesis.** When a device is replaced under warranty or recall, the credit received from the manufacturer
must reduce the claim - reported through the relevant modifiers and condition codes. Where the
overpayment identification and return workflow does not cover device credits, replacement
procedures are billed at full value with no credit reported, and the resulting retained
overpayment carries False Claims Act exposure under the 60-day rule.

**2 · What to check.** Identify device replacement procedures - a device implant preceded by a removal or explant of
the same device family - and test whether the claim carries a credit modifier or condition
code. Report replacements with no credit reported.

**3 · Reference SQL.**

```sql
-- TEST 16 - Device replacement with no credit reported
-- Credit reporting uses modifiers FB (no cost) and FC (partial credit) and condition
-- codes 49/50 on institutional claims. Confirm current reporting requirements.
-- Params: :period_start, :period_end, :replacement_window_days (30)
-- Portability: date +/- integer days and date - date (a day count) are PostgreSQL/DuckDB style;
--   elsewhere use DATEADD / DATE_ADD for the offset and DATEDIFF / DATE_DIFF for the difference.

WITH replacements AS (
  SELECT
      impl.claim_id, impl.claim_line_id, impl.beneficiary_id,
      impl.billing_npi, impl.facility_npi,
      impl.cpt AS implant_cpt, impl.service_date AS implant_date, impl.paid_amount,
      rem.cpt AS removal_cpt, rem.service_date AS removal_date,
      dev.device_family
  FROM claim_lines impl
  JOIN device_procedure_map dev ON dev.hcpcs = impl.cpt AND dev.procedure_role = 'IMPLANT'
  JOIN claim_lines rem
    ON  rem.beneficiary_id = impl.beneficiary_id
    AND rem.service_date BETWEEN impl.service_date - :replacement_window_days
                             AND impl.service_date
    AND rem.adjudication_status = 'PAID'
  JOIN device_procedure_map dev2
    ON  dev2.hcpcs = rem.cpt
    AND dev2.procedure_role = 'REMOVAL'
    AND dev2.device_family  = dev.device_family
  WHERE impl.service_date BETWEEN :period_start AND :period_end
    AND impl.adjudication_status = 'PAID'
)
SELECT
    r.*,
    CASE WHEN EXISTS (
      SELECT 1 FROM claim_lines c
      WHERE c.claim_id = r.claim_id
        AND (c.modifier_1 IN ('FB','FC') OR c.modifier_2 IN ('FB','FC'))
    ) THEN 1 ELSE 0 END AS has_credit_modifier,
    CASE WHEN EXISTS (
      SELECT 1 FROM claim_condition_code cc
      WHERE cc.claim_id = r.claim_id AND cc.condition_code IN ('49','50')
    ) THEN 1 ELSE 0 END AS has_credit_condition_code
FROM replacements r
WHERE NOT EXISTS (
        SELECT 1 FROM claim_lines c
        WHERE c.claim_id = r.claim_id
          AND (c.modifier_1 IN ('FB','FC') OR c.modifier_2 IN ('FB','FC')))
  AND NOT EXISTS (
        SELECT 1 FROM claim_condition_code cc
        WHERE cc.claim_id = r.claim_id AND cc.condition_code IN ('49','50'))
ORDER BY r.paid_amount DESC;
```

**4 · Threshold guidance.** Categorical once a replacement is correctly identified - and identifying the replacement is the
hard part, since it depends entirely on `device_procedure_map` correctly pairing implant and
removal codes within a device family. Build that mapping before the first run. Not every
replacement generates a credit: devices outside warranty, replacements for a changed clinical
indication, and upgrades all legitimately carry no credit, so the output is a question about
whether a credit was received, answerable only from the provider's own purchasing records. That
makes this a provider-inquiry list rather than an overpayment list. Tune to your population; a
starting point, not a standard.

**5 · Paired control + failure mode.** PI-07 (60-day self-identified overpayment report-&-return workflow). Secondary: CB-17 (Prepayment edit maintenance & tuning), PI-09 (Overpayment recovery tracking). See gap note G3 - no control names device credit reporting.

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No workflow identifies device-credit overpayments. PI-07's refund and 60-day timestamp do not exist for this category. |
| **Designed-not-operating** | The 60-day workflow exists generally but has never been applied to device credits. |
| **Operated-but-bypassed** | Credits were received and the corresponding claim adjustment was not made within the 60-day window. |
| **Operated-but-ineffective** | The workflow relies on self-identification with no analytic trigger, so a credit that nobody flags never enters it. |
| **Evidence-missing** | Adjustments were made but no record ties the refund to the credit that prompted it or evidences the 60-day timing. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## Gap notes - controls this family needs that the library does not yet have

*Each note names a control objective these tests point to that the control library does not yet state. They are candidates for a future library release; the library itself is unchanged by this pack.*

### G1. No clinical-validation controls - the gap bites hardest in this family.

Also raised against the coding and billing family and restated here because it is more
consequential at high cost. Every test in this family that touches medical necessity (4, 5, 6)
pairs to a documentation or coverage-criteria control - CD-03, CB-11, CD-01 - which ask whether
documentation exists and whether criteria were applied. None asks whether the documented
clinical picture actually supports the coded diagnosis. That is clinical validation, and for
sepsis, encephalopathy and malnutrition (three heavily audited DRG groups) the control library
is silent.

**Recommendation.** Clinical-validation controls are a candidate for Modules 03 and 05. Once those controls exist,
the paired analytics can be added: sepsis DRG coded without supporting lactate, culture or
vasopressor claims; malnutrition coded without a dietitian assessment or nutritional support.
Until then, tests 4–6 are the closest available proxies and should not be described as clinical
validation.

### G2. No control names physician-owned distributorships or the spinal-implant supply chain.

PI-23 (MSO/investment-disguised kickback detection) is the nearest control and is written
around management-services organisations and Open Payments. Physician-owned distributorships in
spinal implants are a distinct, long-standing and specifically-warned-about arrangement in
exactly the highest-cost procedure family in the toolkit, and test 13 is paired to PI-23 by
proximity rather than by design.

**Recommendation.** Candidate for Module 01 (Provider Enrollment, alongside the ownership-disclosure controls
PE-03/PE-04/PE-05) or Module 02: a device-supply-chain ownership control testing whether
implant-supplier ownership interests held by referring or operating physicians are captured,
disclosed and monitored against referral volume. It extends the existing beneficial-ownership
work rather than starting fresh.

### G3. No control names device credit and warranty-replacement reporting.

Test 16 is paired to PI-07, the generic 60-day overpayment workflow, because no control
addresses credit reporting specifically. The reporting mechanism (modifiers FB/FC, condition
codes 49/50) is well established and the failure is recurrent in audit findings, but nothing in
the library prompts an assessor to test it.

**Recommendation.** Small candidate control for Module 03 or Module 02: device-credit identification and claim-
reduction control, with the analytic trigger (implant preceded by same-family removal, no
credit modifier or condition code) stated in its design so it does not depend on self-
identification.

### G4. CB-13 covers place-of-service coding accuracy, not site-of-service utilisation shift.

Test 12 pairs to CB-13 because it is the only setting-related control, but the two address
different risks. CB-13 asks whether the rate paid matches the POS asserted - a per-claim
correctness question. Nothing asks whether the mix of settings is migrating toward the higher-
paying one over time, which is a pattern question and invisible to a per-claim edit by
construction.

**Recommendation.** Either extend CB-13's design to include a periodic setting-mix monitoring requirement, or
author a companion detective control in Module 03. The distinction is worth preserving
explicitly, because 'every claim was coded correctly' is the exact defence a migration pattern
attracts, and it is a valid one against CB-13.

---

*Reference patterns. Public-source, personal-capacity, advisory and organizational only. Queries must be adapted to your own schema and tested on your own warehouse before use. Outputs are review candidates for human adjudication, never payment or denial dispositions. Confirm every regulatory anchor against its primary source.*