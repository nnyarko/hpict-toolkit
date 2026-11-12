# Analytics Test Pack - Coding & billing integrity - E&M level distribution and modifier overuse

*upcoding, edit-bypass modifiers, and the code-level integrity of a paid claim*

*Generated 2026-11-11 from the HPI-CT control library (`data/controls.csv`, 173 controls). 16 detection tests, paired to 16 primary controls. Reference patterns, not programs.*

## Before you run anything

**These queries produce review candidates, never payment or denial decisions.** Every hit is a question for a human.

**The toolkit never receives your data.** These are patterns you run inside your own environment. No claim data, no PHI, no identifiers leave your systems.

**Every query is a pattern, not a program.** Field names are generic; yours will differ. Expect to rewrite the FROM and JOIN clauses and to remap service categories to your own taxonomy.

**Every threshold is a starting point, not a standard.** Where a threshold is arithmetic it is labelled as such. Everywhere else, measure your own distribution first.

Read `PREFACE_how_to_run_these.md` before the first run.

## Scope of this family

Professional evaluation-and-management coding, the modifiers that bypass correct-coding edits,
units-of-service limits, place-of-service rate selection, inpatient DRG and discharge-status
coding, unbundling, and code-set currency. This is the family the toolkit's control library
covers most densely - Module 03 carries 22 controls - which means nearly every test here pairs
to a control that already exists and already names an owner, a cadence and an evidence
artifact. The analytics layer is the only thing missing.

Coding tests differ from the time-integrity family in one important way: almost nothing here is
arithmetically impossible. A level-5 office visit, a modifier 25, a non-facility place of
service - each is legitimate in isolation, and the signal lives entirely in the distribution.
That makes peer-group construction, not query logic, the hard part, and it makes provider
abrasion a real risk. Every test in this family is a prioritisation signal, and the threshold
guidance says so repeatedly and deliberately.

## Tests at a glance

| # | Test | Primary control | Also touches |
|---|---|---|---|
| 1 | E&M level distribution outlier - high-level share vs specialty peers | **CB-03** | PI-01, CB-19 |
| 2 | Modifier-25 overuse - separately identifiable E&M alongside a same-day procedure | **CB-05** | CB-04, CB-10 |
| 3 | Modifier-59 / X{EPSU} concentration on PTP-edit code pairs | **CB-04** | CB-05, CB-16 |
| 4 | New-patient E&M billed for an established patient | **CB-19** | CB-03, CB-18 |
| 5 | E&M billed inside a global surgical period | **CB-10** | CB-05, CB-09 |
| 6 | Units of service above the MUE ceiling | **CB-07** | CB-08, CB-18 |
| 7 | Place-of-service mismatch - non-facility rate paid for a facility service | **CB-13** | CB-18, CD-18 |
| 8 | Post-acute-transfer DRG - full DRG paid where a transfer occurred | **CB-14** | CB-19, CD-17 |
| 9 | Same-sample UDT code stacking - presumptive plus definitive drug testing | **CB-16** | CB-04, CD-16 |
| 10 | Duplicate professional services across NPIs within one group | **CB-15** | CB-09, CD-17 |
| 11 | Retired, deleted or not-yet-effective code paid | **CB-02** | CB-18, CB-17 |
| 12 | Policy-required diagnosis absent from the claim | **CB-12** | CB-11, PI-04 |
| 13 | Diagnosis-to-service medical-necessity linkage failure (NCD/LCD) | **CB-11** | CD-01, CD-02 |
| 14 | Assistant-at-surgery and co-surgery modifiers on ineligible procedures | **CB-09** | CB-04, CB-19 |
| 15 | Telehealth modality - modifier, place of service and record disagree | **CD-18** | CB-13, CB-17 |
| 16 | Edit-coverage blind spot - high-risk codes with zero prepayment edit activity | **CB-17** | PI-04, CB-21 |

> **Control-ID validation:** every control referenced above resolves to a real ID in the current control library. See the gap notes at the end for controls this family *needs* but the library does not yet have.

---

## 1. E&M level distribution outlier - high-level share vs specialty peers

| | |
|---|---|
| **Paired control** | **CB-03** - E&M level validation vs 2021 MDM/time; peer-benchmark for upcoding |
| **Control type** | Detective |
| **Module** | Module 03 - Coding & Billing Integrity |
| **Owner** | Coding auditor / PI analyst |
| **Frequency** | Monthly |
| **System touchpoint** | Claims analytics |
| **Evidence artifact** | E&M distribution / outlier report |
| **Control failure mode** | Systematic E&M upcoding (levels 4-5) |
| **Regulatory anchor** | CMS Evaluation & Management (2021 guidelines) [guidance]; Program Integrity Alliance - GovQuery / PIA Connect (MCP) [case evidence] |
| **Secondary controls** | PI-01 - Routine claims-aberrancy analytics (peer-outlier / statistical); CB-19 - Coding audit program (statistically valid sampling) + error-rate … |

**1 · Data hypothesis.** Where E&M level validation against the 2021 MDM/time guidelines is absent or runs only as a
chart audit with no distribution view, systematic upcoding is invisible at the claim level and
obvious at the provider level. Each individual level-4 or level-5 visit is defensible; the
provider whose panel is 85% level-4/5 against a peer mean of 40% is the finding. This is the
canonical peer-benchmark test.

**2 · What to check.** For each rendering provider, compute the share of office/outpatient E&M visits coded at the two
highest levels, compute the mean and standard deviation of that share within a specialty peer
group, and report providers above a z-score cutoff. Separate new-patient (99202–99205) from
established-patient (99211–99215) distributions - mixing them distorts the share for any
provider with an unusual new-patient mix.

**3 · Reference SQL.**

```sql
-- TEST 1 - E&M level distribution outlier (upcoding)
-- Carries a new/established split, a minimum-volume floor, and a minimum
-- peer-group size.
-- Params: :period_start, :period_end, :z_threshold (2.5), :min_visits (50), :min_peer_n (15)

WITH em_lines AS (
  SELECT
      cl.rendering_npi,
      pr.specialty,
      pr.peer_region,
      CASE WHEN cl.cpt BETWEEN '99202' AND '99205' THEN 'NEW'
           WHEN cl.cpt BETWEEN '99211' AND '99215' THEN 'ESTABLISHED'
      END AS visit_class,
      CASE WHEN cl.cpt IN ('99204','99205','99214','99215') THEN 1.0 ELSE 0 END AS is_high_level,
      cl.paid_amount
  FROM claim_lines cl
  JOIN provider pr ON pr.npi = cl.rendering_npi
  WHERE cl.service_date BETWEEN :period_start AND :period_end
    AND cl.adjudication_status = 'PAID'
    AND (cl.cpt BETWEEN '99202' AND '99205' OR cl.cpt BETWEEN '99211' AND '99215')
),
prov AS (
  SELECT
      rendering_npi, specialty, peer_region, visit_class,
      COUNT(*)                AS visits,
      AVG(is_high_level)      AS high_em_share,
      SUM(paid_amount)        AS paid_amount
  FROM em_lines
  WHERE visit_class IS NOT NULL
  GROUP BY rendering_npi, specialty, peer_region, visit_class
  HAVING COUNT(*) >= :min_visits
),
peer AS (
  SELECT
      specialty, peer_region, visit_class,
      AVG(high_em_share)     AS peer_mean,
      STDDEV(high_em_share)  AS peer_sd,
      COUNT(*)               AS peer_n
  FROM prov
  GROUP BY specialty, peer_region, visit_class
  HAVING COUNT(*) >= :min_peer_n            -- a peer group of four is not a peer group
)
SELECT
    p.rendering_npi, p.specialty, p.peer_region, p.visit_class,
    p.visits,
    ROUND(p.high_em_share, 3) AS high_em_share,
    ROUND(k.peer_mean, 3)     AS peer_mean,
    ROUND((p.high_em_share - k.peer_mean) / NULLIF(k.peer_sd, 0), 2) AS z_score,
    k.peer_n,
    p.paid_amount,
    -- exposure if the provider coded at the peer mean instead
    ROUND(p.paid_amount * (p.high_em_share - k.peer_mean) / NULLIF(p.high_em_share, 0), 0)
      AS indicative_delta_dollars
FROM prov p
JOIN peer k USING (specialty, peer_region, visit_class)
WHERE (p.high_em_share - k.peer_mean) / NULLIF(k.peer_sd, 0) > :z_threshold
ORDER BY z_score DESC;

-- indicative_delta_dollars is an ORDERING aid, not an overpayment estimate. Do not
-- put it in front of a provider. Extrapolation requires a valid statistical sample
-- (control PI-05), not a z-score.
```

**4 · Threshold guidance.** :z_threshold = 2.5 is the conventional starting cut. Two other parameters matter more:
:min_visits = 50 keeps the share stable, and :min_peer_n = 15 keeps the peer statistics
meaningful. The high-E&M share distribution is bounded at 0 and 1 and is usually not normal, so
a z-score is a convenience, not a claim about probability - consider a raw 95th-percentile cut
or a logit transform before defending the number to a provider. Peer group definition drives
everything: too broad and genuine outliers regress to the mean, too narrow and every provider
is an outlier in a group of three. Tune to your population; a starting point, not a standard.

**5 · Paired control + failure mode.** CB-03 (E&M level validation vs 2021 MDM/time; peer-benchmark for upcoding). Secondary: PI-01 (Routine claims-aberrancy analytics), CB-19 (Coding audit program with statistically valid sampling).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No E&M distribution or outlier report exists. CB-03's named evidence artifact - the E&M distribution / outlier report - does not exist. |
| **Designed-not-operating** | CB-03 is specified monthly; no scan was run for one or more months in the period. Establish what was exposed in the interval. |
| **Operated-but-bypassed** | Outliers were identified and no chart validation, education (PI-12) or edit change followed. |
| **Operated-but-ineffective** | The benchmark uses a national or all-specialty comparison group rather than a like-for-like peer group, so the provider's own specialty norm is never the reference. |
| **Evidence-missing** | The distribution exists in a BI tool but no dated report and no reviewer disposition are retained. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 2. Modifier-25 overuse - separately identifiable E&M alongside a same-day procedure

| | |
|---|---|
| **Paired control** | **CB-05** - Modifier-59 / X{EPSU} governance & documentation review |
| **Control type** | Detective |
| **Module** | Module 03 - Coding & Billing Integrity |
| **Owner** | Coding auditor |
| **Frequency** | Monthly |
| **System touchpoint** | Claim edit engine |
| **Evidence artifact** | Modifier-use audit sample |
| **Control failure mode** | Modifier abuse to defeat PTP edits |
| **Regulatory anchor** | CMS National Correct Coding Initiative (NCCI) - PTP edits & Policy Manual [guidance] |
| **Secondary controls** | CB-04 - NCCI PTP edit enforcement pre-submission; CB-10 - Global-surgical-package bundling control |

**1 · Data hypothesis.** Modifier 25 asserts that a significant, separately identifiable E&M service was performed on
the same day as a procedure. Where modifier governance is absent or reviews only modifier 59, a
provider can append 25 as a default and collect an E&M on top of every procedure. The signature
is a modifier-25 attachment rate far above the peer norm - and, in the strongest cases, a rate
at or near 100%, which is a statement that no procedure the provider ever performs is routine.

**2 · What to check.** For each rendering provider, compute the share of same-day E&M-plus-procedure encounters
carrying modifier 25, benchmark within specialty, and flag providers above the cutoff. Report
the near-100% cohort separately - it needs no statistics.

**3 · Reference SQL.**

```sql
-- TEST 2 - Modifier-25 attachment rate vs specialty peers
-- Modifier 25 = significant, separately identifiable E&M by the same physician on the
-- same day as a procedure. NCCI PTP edits and the NCCI Policy Manual govern which
-- pairs it may bypass: https://www.cms.gov/medicare/coding-billing/ncci-medicaid
-- Params: :period_start, :period_end, :z_threshold (2.5), :min_encounters (30), :min_peer_n (15), :saturation_rate (0.95)

WITH same_day AS (
  -- encounters where an E&M and a procedure were billed for the same beneficiary/provider/day
  SELECT
      em.rendering_npi,
      em.beneficiary_id,
      em.service_date,
      MAX(CASE WHEN em.modifier_1 = '25' OR em.modifier_2 = '25'
                 OR em.modifier_3 = '25' OR em.modifier_4 = '25'
               THEN 1 ELSE 0 END) AS has_mod25,
      SUM(em.paid_amount)         AS em_paid
  FROM claim_lines em
  WHERE em.service_date BETWEEN :period_start AND :period_end
    AND em.adjudication_status = 'PAID'
    AND (em.cpt BETWEEN '99202' AND '99215' OR em.cpt BETWEEN '99281' AND '99285')
    AND EXISTS (
      SELECT 1 FROM claim_lines px
      WHERE px.beneficiary_id = em.beneficiary_id
        AND px.rendering_npi  = em.rendering_npi
        AND px.service_date   = em.service_date
        AND px.cpt BETWEEN '10021' AND '69990'      -- surgical/procedure range; adapt
        AND px.adjudication_status = 'PAID'
    )
  GROUP BY em.rendering_npi, em.beneficiary_id, em.service_date
),
prov AS (
  SELECT
      s.rendering_npi, pr.specialty, pr.peer_region,
      COUNT(*)          AS same_day_encounters,
      AVG(s.has_mod25)  AS mod25_rate,
      SUM(s.em_paid)    AS em_paid_on_same_day
  FROM same_day s
  JOIN provider pr ON pr.npi = s.rendering_npi
  GROUP BY s.rendering_npi, pr.specialty, pr.peer_region
  HAVING COUNT(*) >= :min_encounters
),
peer AS (
  SELECT specialty, peer_region,
         AVG(mod25_rate) AS peer_mean, STDDEV(mod25_rate) AS peer_sd, COUNT(*) AS peer_n
  FROM prov GROUP BY specialty, peer_region HAVING COUNT(*) >= :min_peer_n
)
SELECT
    p.rendering_npi, p.specialty, p.peer_region,
    p.same_day_encounters,
    ROUND(p.mod25_rate, 3) AS mod25_rate,
    ROUND(k.peer_mean, 3)  AS peer_mean,
    ROUND((p.mod25_rate - k.peer_mean) / NULLIF(k.peer_sd, 0), 2) AS z_score,
    p.em_paid_on_same_day,
    CASE WHEN p.mod25_rate >= :saturation_rate THEN 'SATURATED' ELSE 'OUTLIER' END AS class
FROM prov p
JOIN peer k USING (specialty, peer_region)
WHERE p.mod25_rate >= :saturation_rate
   OR (p.mod25_rate - k.peer_mean) / NULLIF(k.peer_sd, 0) > :z_threshold
ORDER BY p.mod25_rate DESC;

-- Companion: the same test for modifier 57 (decision for surgery) and for E&M
-- appended to a MINOR procedure only (global period 000/010), where the separately
-- identifiable claim is hardest to sustain.
```

**4 · Threshold guidance.** :saturation_rate = 0.95 identifies providers appending modifier 25 to essentially every
eligible encounter - that cohort needs no statistical argument and should be worked first.
:z_threshold = 2.5 handles the rest. Specialty matters enormously here: dermatology,
ophthalmology and urgent care legitimately carry high modifier-25 rates because same-day minor
procedures are their normal practice, so a cross-specialty comparison will produce a list of
specialties rather than a list of providers. The CPT ranges in the query are illustrative - set
the procedure range and the global-period filter from your own fee schedule. Tune to your
population; a starting point, not a standard.

**5 · Paired control + failure mode.** CB-05 (Modifier governance & documentation review - its trigger is 'modifier appended to bypass an edit', which is exactly modifier 25, though the control's name currently reads 59/X{EPSU} only; see gap note G1). Secondary: CB-04 (NCCI PTP edit enforcement pre-submission), CB-10 (Global-surgical-package bundling control).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No modifier-use audit exists. CB-05's evidence artifact - the modifier-use audit sample - does not exist. |
| **Designed-not-operating** | CB-05 is specified monthly; no sample was drawn or reviewed in the period. |
| **Operated-but-bypassed** | Modifier-25 exceptions were reviewed and cleared without inspecting whether the record documents a significant, separately identifiable service. |
| **Operated-but-ineffective** | The audit scope covers modifier 59 and X{EPSU} only, so modifier 25 - the highest-volume bypass modifier - is never sampled. This is the defining defect for this test. |
| **Evidence-missing** | Sampling occurred but no retained workpaper ties a sampled claim to a documented conclusion. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 3. Modifier-59 / X{EPSU} concentration on PTP-edit code pairs

| | |
|---|---|
| **Paired control** | **CB-04** - NCCI PTP edit enforcement pre-submission |
| **Control type** | Preventive |
| **Module** | Module 03 - Coding & Billing Integrity |
| **Owner** | Billing system owner |
| **Frequency** | Per claim |
| **System touchpoint** | Claim scrubber / edit engine |
| **Evidence artifact** | PTP edit hit / disposition log |
| **Control failure mode** | Payment for improperly bundled code pairs |
| **Regulatory anchor** | CMS National Correct Coding Initiative (NCCI) - PTP edits & Policy Manual [guidance]; context only: Program Integrity Alliance, Prioritizing Prevention: Value-for-Money in Anti-Fraud Efforts (white paper, 1 Mar 2025), argues prevention should be credited alongside recoveries and does not address NCCI [case evidence] |
| **Secondary controls** | CB-05 - Modifier-59 / X{EPSU} governance & documentation review; CB-16 - Unbundling detection analytics - incl. same-sample … |

**1 · Data hypothesis.** NCCI procedure-to-procedure edits bundle code pairs that should not normally be billed
together; modifier 59 and the X{EPSU} subset are the permitted bypass. Where PTP enforcement is
absent, or enforced but never monitored for bypass volume, a provider can restore every bundled
pair by reflex. The signature is a high bypass rate on the specific code pairs that carry a PTP
edit - not modifier 59 volume generally.

**2 · What to check.** Join paid claim lines to your NCCI PTP edit file on the (column-1, column-2) code pair, and for
each provider compute the share of edit-eligible pairs that were paid because a 59/X modifier
was present. Report the providers, and separately the code pairs, with the highest bypass
rates.

**3 · Reference SQL.**

```sql
-- TEST 3 - PTP-edit bypass rate via modifier 59 / X{EPSU}
-- Requires the NCCI PTP edit file loaded as a reference table. Medicaid NCCI files
-- (six methodologies, quarterly): https://www.cms.gov/medicare/coding-billing/ncci-medicaid
-- Params: :period_start, :period_end, :min_pairs (25), :bypass_rate_cut (0.50)

WITH paid_pairs AS (
  SELECT
      a.rendering_npi,
      a.beneficiary_id,
      a.service_date,
      a.cpt AS col1_code,
      b.cpt AS col2_code,
      CASE WHEN b.modifier_1 IN ('59','XE','XP','XS','XU')
             OR b.modifier_2 IN ('59','XE','XP','XS','XU')
             OR b.modifier_3 IN ('59','XE','XP','XS','XU')
             OR b.modifier_4 IN ('59','XE','XP','XS','XU')
           THEN 1 ELSE 0 END AS bypass_modifier_present,
      b.paid_amount
  FROM claim_lines a
  JOIN claim_lines b
    ON  b.beneficiary_id = a.beneficiary_id
    AND b.rendering_npi  = a.rendering_npi
    AND b.service_date   = a.service_date
    AND b.claim_line_id <> a.claim_line_id
  JOIN ncci_ptp_edit e                       -- your loaded NCCI PTP reference table
    ON  e.column_1_code = a.cpt
    AND e.column_2_code = b.cpt
    AND e.modifier_indicator = '1'           -- '1' = a modifier may bypass this edit
    AND a.service_date BETWEEN e.effective_date
                           AND COALESCE(e.deletion_date, DATE '9999-12-31')
  WHERE a.service_date BETWEEN :period_start AND :period_end
    AND a.adjudication_status = 'PAID'
    AND b.adjudication_status = 'PAID'
),
by_provider AS (
  SELECT
      rendering_npi,
      COUNT(*)                       AS edit_eligible_pairs,
      AVG(bypass_modifier_present)   AS bypass_rate,
      SUM(CASE WHEN bypass_modifier_present = 1 THEN paid_amount ELSE 0 END) AS bypassed_paid
  FROM paid_pairs
  GROUP BY rendering_npi
  HAVING COUNT(*) >= :min_pairs
)
SELECT
    p.rendering_npi, pr.specialty,
    p.edit_eligible_pairs,
    ROUND(p.bypass_rate, 3) AS bypass_rate,
    p.bypassed_paid
FROM by_provider p
JOIN provider pr ON pr.npi = p.rendering_npi
WHERE p.bypass_rate > :bypass_rate_cut
ORDER BY p.bypass_rate DESC, p.bypassed_paid DESC;

-- Second cut, by CODE PAIR rather than provider: which pairs are bypassed most often
-- across the whole population? A pair bypassed at 90% network-wide is usually an edit
-- or policy problem, not a provider problem - and belongs to CB-17, not to an audit.
```

**4 · Threshold guidance.** :bypass_rate_cut = 0.50 is a deliberately blunt starting cut - half of all edit-eligible pairs
restored by modifier is high in most specialties, but there are legitimate exceptions (distinct
anatomic sites in dermatology and podiatry, separate encounters in the same day). :min_pairs =
25 keeps the rate stable. Run the by-code-pair cut FIRST: if a pair is bypassed at high rates
network-wide, you have found a policy or edit-configuration issue and the provider list derived
from it will be noise. Tune to your population; a starting point, not a standard.

**5 · Paired control + failure mode.** CB-04 (NCCI PTP edit enforcement pre-submission). Secondary: CB-05 (Modifier-59 / X{EPSU} governance & documentation review), CB-16 (Unbundling detection analytics).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | PTP edits are not enforced pre-submission. CB-04's PTP edit hit/disposition log does not exist. |
| **Designed-not-operating** | The edit engine is configured but the edit file was not refreshed for the period - CB-06 requires quarterly Medicaid NCCI file loads. |
| **Operated-but-bypassed** | PTP hits were overridden at adjudication to clear a claims backlog, with no modifier-documentation review. |
| **Operated-but-ineffective** | Edits fire and the bypass modifier is accepted at face value with no downstream monitoring of bypass volume - the edit runs, the control does not work. |
| **Evidence-missing** | Hits are suppressed silently in the edit engine with no retained disposition log. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 4. New-patient E&M billed for an established patient

| | |
|---|---|
| **Paired control** | **CB-19** - Coding audit program (statistically valid sampling) + error-rate tracking |
| **Control type** | Detective |
| **Module** | Module 03 - Coding & Billing Integrity |
| **Owner** | Coding compliance / internal audit |
| **Frequency** | Quarterly |
| **System touchpoint** | Audit workpaper system |
| **Evidence artifact** | Coding-audit report + error rate |
| **Control failure mode** | Undetected systematic coding error |
| **Regulatory anchor** | CMS Pub 100-08 Program Integrity Manual [guidance]; OIG A-09-22-03010 (21 Jun 2023) - molecular pathology (CPT 81408); all three recommendations closed [case evidence] |
| **Secondary controls** | CB-03 - E&M level validation vs 2021 MDM/time; peer-benchmark for upcoding; CB-18 - Pre-submission claim scrubber (provider side) |

**1 · Data hypothesis.** New-patient E&M codes pay materially more than their established-patient equivalents and are
defined by a bright-line rule: no face-to-face service from the same physician, or another
physician of the same specialty in the same group, within the prior three years. Where the
coding audit program does not test this specific rule, it goes unchecked, because no edit fires
and no chart looks wrong.

**2 · What to check.** For each new-patient E&M claim line, look back three years for any face-to-face professional
service for the same beneficiary from the same provider - or from another provider sharing the
same group TIN and specialty - and flag hits.

**3 · Reference SQL.**

```sql
-- TEST 4 - New-patient E&M where a prior encounter exists within the lookback
-- CPT defines "new patient" as no face-to-face service from the same physician, or
-- another physician of the same specialty in the same group practice, within 3 years.
-- Params: :period_start, :period_end   (the lookback is three calendar years, not a day count)

SELECT
    n.claim_line_id, n.beneficiary_id, n.rendering_npi, n.group_tin,
    n.cpt, n.service_date, n.paid_amount,
    prior.prior_service_date,
    prior.prior_rendering_npi,
    (n.service_date - prior.prior_service_date) AS days_since_prior,
    CASE WHEN prior.prior_rendering_npi = n.rendering_npi
         THEN 'SAME_PROVIDER' ELSE 'SAME_GROUP_AND_SPECIALTY' END AS match_basis
FROM claim_lines n
JOIN provider pn ON pn.npi = n.rendering_npi
CROSS JOIN LATERAL (                      -- Postgres/Snowflake; use OUTER APPLY on SQL Server
    SELECT p.service_date AS prior_service_date,
           p.rendering_npi AS prior_rendering_npi
    FROM claim_lines p
    JOIN provider pp ON pp.npi = p.rendering_npi
    WHERE p.beneficiary_id = n.beneficiary_id
      AND p.service_date < n.service_date
      -- interval syntax is PostgreSQL and DuckDB; elsewhere use DATEADD(year, -3, n.service_date)
      AND p.service_date >= n.service_date - INTERVAL '3 years'   -- calendar years (1096 days across a leap day)
      AND p.adjudication_status = 'PAID'
      AND p.is_face_to_face = TRUE                    -- derive from POS + code range
      AND (p.rendering_npi = n.rendering_npi
           OR (pp.group_tin = pn.group_tin AND pp.specialty = pn.specialty))
    -- ordering by a boolean is PostgreSQL and DuckDB; elsewhere use
    -- ORDER BY CASE WHEN p.rendering_npi = n.rendering_npi THEN 0 ELSE 1 END
    ORDER BY (p.rendering_npi = n.rendering_npi) DESC,   -- same-provider prior first
             p.service_date DESC
    LIMIT 1
) prior
WHERE n.service_date BETWEEN :period_start AND :period_end
  AND n.adjudication_status = 'PAID'
  AND n.cpt IN ('99202','99203','99204','99205')      -- add 99341-99350, 99381-99387 as applicable
ORDER BY n.paid_amount DESC;

-- Roll up by rendering_npi and by group_tin. A single hit is a keying error;
-- a provider with a 30% new-patient-conflict rate is a control finding.
```

**4 · Threshold guidance.** The three-year rule is definitional, not tunable - but two things around it are. First,
`is_face_to_face`: telehealth, lab-only and interpretation-only encounters do not reset new-
patient status in the same way, so how you derive that flag determines your false-positive rate
more than anything else. Second, group and specialty resolution: if your provider dimension has
stale TINs or coarse specialty taxonomy, SAME_GROUP_AND_SPECIALTY matches will be unreliable  - 
work SAME_PROVIDER matches first, they need no dimension quality at all. Tune to your
population; a starting point, not a standard.

**5 · Paired control + failure mode.** CB-19 (Coding audit program with statistically valid sampling + error-rate tracking). Secondary: CB-03 (E&M level validation), CB-18 (Pre-submission claim scrubber, provider side). See gap note G2 - no control names the new-vs-established rule specifically.

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No coding audit program covers the new-patient rule, and no scrubber edit tests it. |
| **Designed-not-operating** | CB-19 is specified quarterly; no audit cycle covering E&M ran in the period. |
| **Operated-but-bypassed** | The audit identified new-patient errors and no corrective-action or coder-education loop followed (CB-20). |
| **Operated-but-ineffective** | The audit samples E&M level accuracy only and never tests new-vs-established status, so the error class is structurally out of scope. |
| **Evidence-missing** | Audit occurred but the workpaper does not record which attributes were tested. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 5. E&M billed inside a global surgical period

| | |
|---|---|
| **Paired control** | **CB-10** - Global-surgical-package bundling control |
| **Control type** | Detective |
| **Module** | Module 03 - Coding & Billing Integrity |
| **Owner** | Coding auditor |
| **Frequency** | Monthly |
| **System touchpoint** | Claims analytics |
| **Evidence artifact** | Global-period conflict report |
| **Control failure mode** | Separate billing of bundled post-op care |
| **Regulatory anchor** | CMS Pub 100-04 Claims Processing Manual [guidance] |
| **Secondary controls** | CB-05 - Modifier-59 / X{EPSU} governance & documentation review; CB-09 - Payment-modifier validation (co-surgery 62, assistant 80/AS, … |

**1 · Data hypothesis.** Major and minor surgical procedures carry a global period during which routine post-operative
E&M is bundled into the surgical payment. Where the global-surgical-package control is absent,
or checks only the surgeon's own claims, post-operative visits are billed separately  - 
sometimes with modifier 24 asserting they are unrelated, which is exactly the assertion that
needs testing.

**2 · What to check.** For each paid surgical procedure, look forward across its global period for E&M claims from the
same provider or group, and flag those paid without an appropriate modifier - and, separately,
those paid WITH modifier 24 or 79, which are the assertions to sample.

**3 · Reference SQL.**

```sql
-- TEST 5 - E&M inside the global surgical period
-- Global-period days come from the Medicare Physician Fee Schedule relative value file
-- (000 / 010 / 090 / XXX / YYY / ZZZ). Load it as a reference dimension.
-- Params: :period_start, :period_end

WITH surgeries AS (
  SELECT
      s.claim_line_id AS surgery_line_id,
      s.beneficiary_id, s.rendering_npi, s.group_tin,
      s.cpt AS surgical_cpt, s.service_date AS surgery_date,
      CAST(f.global_period_days AS INT) AS global_days
  FROM claim_lines s
  JOIN fee_schedule f ON f.hcpcs = s.cpt
                      AND f.effective_year = EXTRACT(YEAR FROM s.service_date)   -- one row per code and year
  WHERE s.adjudication_status = 'PAID'
    AND f.global_period_days IN ('010','090')       -- minor with follow-up, and major
    AND s.service_date BETWEEN :period_start - 90 AND :period_end
)
SELECT
    g.surgery_line_id, g.beneficiary_id, g.rendering_npi, g.surgical_cpt,
    g.surgery_date, g.global_days,
    e.claim_line_id AS em_line_id, e.cpt AS em_cpt, e.service_date AS em_date,
    (e.service_date - g.surgery_date) AS days_into_global,
    e.paid_amount,
    COALESCE(e.modifier_1, '') || COALESCE(e.modifier_2, '')
      || COALESCE(e.modifier_3, '') || COALESCE(e.modifier_4, '') AS em_modifiers,
    CASE
      WHEN e.modifier_1 IN ('24','79') OR e.modifier_2 IN ('24','79')
        OR e.modifier_3 IN ('24','79') OR e.modifier_4 IN ('24','79')
        THEN 'UNRELATED_ASSERTED'      -- sample these: the assertion is the finding
      WHEN e.modifier_1 IN ('58','78')  OR e.modifier_2 IN ('58','78')
        OR e.modifier_3 IN ('58','78') OR e.modifier_4 IN ('58','78')
        THEN 'STAGED_OR_RETURN'        -- usually legitimate; lower priority
      ELSE 'NO_MODIFIER'               -- should not have paid
    END AS global_class
FROM surgeries g
JOIN claim_lines e
  ON  e.beneficiary_id = g.beneficiary_id
  AND (e.rendering_npi = g.rendering_npi OR e.group_tin = g.group_tin)
  AND e.service_date >  g.surgery_date
  AND e.service_date <= g.surgery_date + g.global_days
  AND e.cpt BETWEEN '99202' AND '99215'
  AND e.adjudication_status = 'PAID'
WHERE e.service_date BETWEEN :period_start AND :period_end
ORDER BY global_class, e.paid_amount DESC;
```

**4 · Threshold guidance.** NO_MODIFIER hits are categorical - a bundled visit was paid separately, and the edit should
have caught it. UNRELATED_ASSERTED (modifier 24 or 79) is the interesting bucket and needs a
rate, not a list: compute each provider's share of in-global E&M carrying modifier 24 and
benchmark it within specialty, as in TEST 2. Note the query looks back 90 days before
:period_start so that surgeries preceding the window still have their global periods evaluated
 -  omitting that truncates the first quarter of results. Tune to your population; a starting
point, not a standard.

**5 · Paired control + failure mode.** CB-10 (Global-surgical-package bundling control). Secondary: CB-05 (Modifier governance & documentation review), CB-09 (Payment-modifier validation).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No global-period conflict report exists - CB-10's named evidence artifact. |
| **Designed-not-operating** | CB-10 is specified monthly; no conflict report was produced. |
| **Operated-but-bypassed** | Conflicts were resolved by accepting modifier 24 without documentation review. |
| **Operated-but-ineffective** | The check matches on rendering NPI only, so a partner in the same group billing the post-op visit never conflicts. |
| **Evidence-missing** | Report produced, no retained dispositions. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 6. Units of service above the MUE ceiling

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
| **Secondary controls** | CB-08 - Quantity / frequency-limit edits (capped & time-based services); CB-18 - Pre-submission claim scrubber (provider side) |

**1 · Data hypothesis.** Medically Unlikely Edits cap the units of a code that could plausibly be furnished to one
beneficiary on one day. Where MUE enforcement is absent, stale, or applied per line while the
billing is split across lines, over-unit claims pay. The line-splitting variant is the one that
survives most edit engines.

**2 · What to check.** Compare billed units to the MUE value for the code, at BOTH line level and date-of-service
aggregate level, respecting the MUE adjudication indicator (MAI) which determines whether the
edit is per line or per date of service.

**3 · Reference SQL.**

```sql
-- TEST 6 - MUE breach at line level and at date-of-service level
-- MUE values and MAI indicators come from the CMS NCCI MUE files (Practitioner,
-- Facility Outpatient, DME). MAI 1 = per line; MAI 2 = per date of service, absolute;
-- MAI 3 = per date of service, "not otherwise specified".
-- Params: :period_start, :period_end, :mue_file_type ('PRACTITIONER', 'FACILITY_OP' or 'DME')

WITH line_level AS (
  SELECT
      cl.claim_line_id, cl.claim_id, cl.beneficiary_id, cl.rendering_npi, cl.billing_npi,
      cl.cpt, cl.service_date, cl.units, cl.paid_amount,
      m.mue_value, m.mai
  FROM claim_lines cl
  JOIN ncci_mue m
    ON  m.hcpcs = cl.cpt
    AND m.mue_file = :mue_file_type          -- 'PRACTITIONER' | 'FACILITY_OP' | 'DME'
  WHERE cl.service_date BETWEEN :period_start AND :period_end
    AND cl.adjudication_status = 'PAID'
),
dos_level AS (
  SELECT
      beneficiary_id, rendering_npi, cpt, service_date,
      SUM(units)        AS dos_units,
      MAX(mue_value)    AS mue_value,
      MAX(mai)          AS mai,
      COUNT(*)          AS line_count,
      SUM(paid_amount)  AS paid_amount
  FROM line_level
  GROUP BY beneficiary_id, rendering_npi, cpt, service_date
)
-- 6a: single line above the MUE (MAI 1)
SELECT 'LINE_LEVEL_BREACH' AS breach_type,
       claim_line_id, beneficiary_id, rendering_npi, cpt, service_date,
       units AS billed_units, mue_value, units - mue_value AS excess_units,
       1 AS line_count, paid_amount
FROM line_level
WHERE mai = '1' AND units > mue_value

UNION ALL

-- 6b: units split across lines to stay under a per-line cap (MAI 2/3) - the harder case
SELECT 'DOS_LEVEL_BREACH' AS breach_type,
       CAST(NULL AS VARCHAR) AS claim_line_id,
       beneficiary_id, rendering_npi, cpt, service_date,
       dos_units AS billed_units, mue_value, dos_units - mue_value AS excess_units,
       line_count, paid_amount
FROM dos_level
WHERE mai IN ('2','3') AND dos_units > mue_value
ORDER BY excess_units DESC, paid_amount DESC;
```

**4 · Threshold guidance.** Categorical against the published MUE value - but two cautions. MUE values are revised
quarterly, so join on the MUE file version effective for the service date, not the current one,
or you will manufacture historical breaches. And the DOS_LEVEL_BREACH bucket with a high
line_count is the signal worth working: split lines that individually clear a per-line cap and
jointly exceed a date-of-service cap are deliberate in a way a single over-unit line usually is
not. Tune to your population; a starting point, not a standard.

**5 · Paired control + failure mode.** CB-07 (MUE units-of-service edit enforcement). Secondary: CB-08 (Quantity / frequency-limit edits), CB-18 (Pre-submission claim scrubber).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | MUE edits are not enforced. CB-07's MUE hit / adjudication log does not exist. |
| **Designed-not-operating** | The edit exists but the MUE file has not been refreshed through the quarterly cycle. |
| **Operated-but-bypassed** | MUE hits were overridden without documentation of medical necessity for the excess units. |
| **Operated-but-ineffective** | The edit applies per line only and ignores the MAI, so date-of-service caps are never enforced and line-splitting defeats it entirely. |
| **Evidence-missing** | No retained adjudication log of MUE hits and their disposition. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 7. Place-of-service mismatch - non-facility rate paid for a facility service

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
| **Secondary controls** | CB-18 - Pre-submission claim scrubber (provider side); CD-18 - Service-modality documentation control: note and claim modifier … |

**1 · Data hypothesis.** The place-of-service code drives whether a service pays at the higher non-facility rate
(practice bears the overhead) or the lower facility rate. Where the POS control is absent, a
provider can bill POS 11 (office) for a service actually furnished in a facility, collecting
the overhead component twice. The OIG POS-32 finding referenced by CB-13 is the canonical
instance.

**2 · What to check.** For each professional claim paid at the non-facility rate, look for a facility claim for the
same beneficiary on the same date - an inpatient, outpatient, SNF or ASC claim - which
contradicts the asserted place of service.

**3 · Reference SQL.**

```sql
-- TEST 7 - Non-facility POS asserted while a facility claim exists for the same encounter
-- Params: :period_start, :period_end

SELECT
    p.claim_line_id, p.beneficiary_id, p.rendering_npi, p.billing_npi,
    p.cpt, p.service_date, p.place_of_service AS asserted_pos,
    p.paid_amount AS professional_paid,
    f.claim_type  AS contradicting_facility_claim,
    f.facility_npi,
    f.admit_date, f.discharge_date,
    fs.non_facility_rate - fs.facility_rate AS rate_differential
FROM claim_lines p
JOIN fee_schedule fs ON fs.hcpcs = p.cpt
JOIN facility_claim f
  ON  f.beneficiary_id = p.beneficiary_id
  AND (   (p.service_date >  f.admit_date            -- strictly inside the stay: admission and
           AND p.service_date <  COALESCE(f.discharge_date, DATE '9999-12-31'))  -- discharge days excluded; open stay runs on
       OR (f.claim_type IN ('OUTPATIENT','ASC')       -- same-day outpatient or ASC encounter: the
           AND f.discharge_date = f.admit_date        -- service date is the encounter itself
           AND p.service_date = f.admit_date))
WHERE p.service_date BETWEEN :period_start AND :period_end
  AND p.adjudication_status = 'PAID'
  AND p.place_of_service IN ('11','49','20')          -- office / independent clinic / urgent care
  AND f.claim_type IN ('INPATIENT','OUTPATIENT','SNF','ASC')
  AND fs.non_facility_rate > fs.facility_rate         -- only where the differential exists
ORDER BY rate_differential DESC, p.paid_amount DESC;

-- Companion, no facility-claim join required: providers whose POS-11 share is far above
-- their specialty peers while their patients' facility utilisation is comparable.
```

**4 · Threshold guidance.** Categorical where a contradicting facility claim exists on the same date - but false positives
are real: a patient can legitimately be seen in an office in the morning and admitted in the
afternoon. Exclude the admission date and the discharge date, as in the home-health family's
institutional-stay test, and rank by rate_differential rather than by paid_amount so you
surface the codes where the POS choice actually moved money. Providers with a recurring pattern
across many beneficiaries are the finding; single instances are usually clinical reality. Tune
to your population; a starting point, not a standard.

**5 · Paired control + failure mode.** CB-13 (Place-of-service / facility-vs-non-facility rate edit). Secondary: CB-18 (Pre-submission claim scrubber), CD-18 (Service-modality documentation control).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No POS-vs-rate exception report exists - CB-13's named evidence artifact. |
| **Designed-not-operating** | CB-13 is specified continuous; no exception report covers the period. |
| **Operated-but-bypassed** | Exceptions were closed on the provider's assertion with no cross-check against the facility claim. |
| **Operated-but-ineffective** | The edit validates that the POS code is syntactically valid rather than that it is consistent with the beneficiary's other claims that day. |
| **Evidence-missing** | No retained exception report or dispositions. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 8. Post-acute-transfer DRG - full DRG paid where a transfer occurred

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
| **Secondary controls** | CB-19 - Coding audit program (statistically valid sampling) + error-rate …; CD-17 - Coordination-of-benefits / setting documentation control to prevent … |

**1 · Data hypothesis.** Under the post-acute-care transfer policy, a qualifying DRG discharged to certain post-acute
settings pays a per-diem transfer amount rather than the full DRG. Where the discharge-status
control is absent, a discharge-status code of 01 (home) is reported and the full DRG pays, even
though a post-acute claim for the same beneficiary starts within days. This is one of the OIG's
most repeated hospital findings and CB-14 cites it directly.

**2 · What to check.** For inpatient claims in transfer-qualifying DRGs coded with a non-transfer discharge status,
look for a post-acute claim - SNF, home health, IRF, LTCH - starting within the transfer
window, and report the DRG payment at risk.

**3 · Reference SQL.**

```sql
-- TEST 8 - Post-acute-transfer DRG: discharge status contradicted by a post-acute claim
-- Transfer-qualifying DRGs are published annually in the IPPS final rule; load as a
-- reference table with the fiscal year they apply to.
-- Params: :period_start, :period_end, :postacute_window_days (3), :fiscal_year (the IPPS fiscal year of the transfer-DRG list)

SELECT
    i.claim_id             AS inpatient_claim_id,
    i.beneficiary_id, i.facility_npi, i.drg_code,
    i.admit_date, i.discharge_date,
    i.discharge_status_code,
    i.paid_amount          AS drg_paid,
    pa.claim_type          AS postacute_setting,
    pa.service_start_date  AS postacute_start,
    (pa.service_start_date - i.discharge_date) AS days_to_postacute
FROM inpatient_claim i
JOIN transfer_drg t
  ON  t.drg_code    = i.drg_code
  AND t.fiscal_year = :fiscal_year
JOIN post_acute_claim pa
  ON  pa.beneficiary_id      = i.beneficiary_id
  AND pa.service_start_date >= i.discharge_date
  AND pa.service_start_date <= i.discharge_date + :postacute_window_days
WHERE i.discharge_date BETWEEN :period_start AND :period_end
  AND i.adjudication_status = 'PAID'
  AND pa.claim_type IN ('SNF','HOME_HEALTH','IRF','LTCH','PSYCH')
  AND (
        -- discharge statuses that assert NO post-acute transfer
        i.discharge_status_code IN ('01','20','21')
        -- status 06 (home under organised home-health care) is a transfer status for the
        -- special-payment DRG subset, and is consistent with a following home-health claim;
        -- flag it only for a non-special DRG followed by a non-home-health post-acute claim
        OR (i.discharge_status_code = '06'
            AND pa.claim_type <> 'HOME_HEALTH'
            -- boolean column and literal; where there is no boolean type store 0/1 and
            -- use COALESCE(t.special_payment_subset, 0) = 0
            AND COALESCE(t.special_payment_subset, FALSE) = FALSE)
      )
ORDER BY i.paid_amount DESC;

-- Note: discharge status 06 (home under care of an organised home-health service) IS a
-- transfer status for the subset of DRGs on the special-payment list (transfer_drg
-- .special_payment_subset), so it is split out above and evaluated against that list.
```

**4 · Threshold guidance.** :postacute_window_days = 3 matches the conventional transfer window for home health; SNF and
IRF transfers are generally evaluated on same-day or next-day admission. Set the window per
post-acute setting rather than globally. The discharge-status list matters more than the
window: status 06 behaves differently for the special-payment DRG subset and will dominate
false positives if handled as a single bucket. Confirm the transfer-qualifying DRG list against
the IPPS final rule for the correct fiscal year - it changes annually. Tune to your population;
a starting point, not a standard.

**5 · Paired control + failure mode.** CB-14 (DRG & discharge-status coding accuracy review - transfer vs discharge). Secondary: CB-19 (Coding audit program), CD-17 (Coordination-of-benefits / setting documentation control).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No discharge-status audit exists. CB-14's discharge-status audit sample does not exist. |
| **Designed-not-operating** | CB-14 is specified monthly; no sample was drawn in the period. |
| **Operated-but-bypassed** | Discrepancies were identified and no claim adjustment or overpayment return followed (PI-07). |
| **Operated-but-ineffective** | The audit reviews the coded discharge status against the medical record only, and never against the downstream post-acute claim - which is the only evidence that reliably contradicts it. |
| **Evidence-missing** | Audit sample not retained in a form that shows which claims were tested. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 9. Same-sample UDT code stacking - presumptive plus definitive drug testing

| | |
|---|---|
| **Paired control** | **CB-16** - Unbundling detection analytics - incl. same-sample presumptive+definitive UDT code-pair edit (CPT 80307 + HCPCS G0483 billed on one sample/date) |
| **Control type** | Detective |
| **Module** | Module 03 - Coding & Billing Integrity |
| **Owner** | PI data analyst |
| **Frequency** | Monthly |
| **System touchpoint** | Claims analytics |
| **Evidence artifact** | Unbundling pattern alert |
| **Control failure mode** | Revenue inflation via fragmented billing / same-sample UDT code-stacking |
| **Regulatory anchor** | Program Integrity Alliance - GovQuery / PIA Connect (MCP) [case evidence] |
| **Secondary controls** | CB-04 - NCCI PTP edit enforcement pre-submission; CD-16 - Enhanced documentation review for high-cost / high-risk tests and … |

**1 · Data hypothesis.** Urine drug testing is billed as a presumptive screen (CPT 80305–80307) and, where clinically
indicated, a definitive confirmation (HCPCS G0480–G0483). Where unbundling analytics are
absent, both are billed on the same sample as a matter of routine, and the definitive test is
billed at the top tier regardless of how many drug classes were actually confirmed. CB-16 names
this pattern explicitly.

**2 · What to check.** Find same-beneficiary, same-date pairs of a presumptive UDT code with a definitive UDT code,
compute each billing provider's stacking rate (stacked samples over all UDT samples), and list
the pairs only for billers with at least :min_tests samples and a stacking rate above
:stack_rate_cut. Separately compute each provider's share of definitive tests billed at the
highest tier (G0483, 22+ drug classes).

**3 · Reference SQL.**

```sql
-- TEST 9 - Same-sample presumptive + definitive UDT, and top-tier definitive concentration
-- Params: :period_start, :period_end, :min_tests (25), :stack_rate_cut (0.50), :top_tier_share_cut (0.80; read only by the commented 9b)

WITH udt AS (
  SELECT
      cl.beneficiary_id, cl.rendering_npi, cl.billing_npi, cl.service_date,
      cl.cpt, cl.paid_amount,
      CASE WHEN cl.cpt IN ('80305','80306','80307')             THEN 'PRESUMPTIVE'
           WHEN cl.cpt IN ('G0480','G0481','G0482','G0483')     THEN 'DEFINITIVE'
      END AS udt_class,
      CASE WHEN cl.cpt = 'G0483' THEN 1.0 ELSE 0 END AS is_top_tier
  FROM claim_lines cl
  WHERE cl.service_date BETWEEN :period_start AND :period_end
    AND cl.adjudication_status = 'PAID'
    AND cl.cpt IN ('80305','80306','80307','G0480','G0481','G0482','G0483')
)
-- 9a: presumptive + definitive on the same sample/date, reported only for billers whose
-- stacking RATE is the signal (a single stack among many samples is not an assertion that
-- every screen was ambiguous)
, sample AS (                                   -- one row per biller, beneficiary and date
  SELECT billing_npi, beneficiary_id, service_date,
         MAX(CASE WHEN udt_class = 'PRESUMPTIVE' THEN 1 ELSE 0 END) AS has_presumptive,
         MAX(CASE WHEN udt_class = 'DEFINITIVE'  THEN 1 ELSE 0 END) AS has_definitive
  FROM udt
  GROUP BY billing_npi, beneficiary_id, service_date
),
biller_rate AS (
  SELECT billing_npi,
         COUNT(*) AS samples,
         SUM(has_presumptive * has_definitive) AS stacked_samples,
         SUM(has_presumptive * has_definitive) * 1.0 / COUNT(*) AS stack_rate
  FROM sample
  GROUP BY billing_npi
  HAVING COUNT(*) >= :min_tests
     AND SUM(has_presumptive * has_definitive) * 1.0 / COUNT(*) > :stack_rate_cut
)
SELECT
    'SAME_SAMPLE_STACK' AS finding,
    p.beneficiary_id, p.rendering_npi, p.billing_npi, p.service_date,
    p.cpt AS presumptive_cpt, d.cpt AS definitive_cpt,
    p.paid_amount + d.paid_amount AS combined_paid,
    r.samples, r.stacked_samples, r.stack_rate
FROM udt p
JOIN udt d
  ON  d.beneficiary_id = p.beneficiary_id
  AND d.service_date   = p.service_date
  AND d.billing_npi    = p.billing_npi
JOIN biller_rate r ON r.billing_npi = p.billing_npi
WHERE p.udt_class = 'PRESUMPTIVE'
  AND d.udt_class = 'DEFINITIVE'
ORDER BY r.stack_rate DESC, combined_paid DESC;

-- 9b: providers whose definitive testing is almost entirely top-tier
--   SELECT rendering_npi, COUNT(*) AS definitive_tests, AVG(is_top_tier) AS top_tier_share,
--          SUM(paid_amount) AS paid
--   FROM udt WHERE udt_class = 'DEFINITIVE'
--   GROUP BY rendering_npi
--   HAVING COUNT(*) >= :min_tests AND AVG(is_top_tier) > :top_tier_share_cut
--   ORDER BY top_tier_share DESC;
```

**4 · Threshold guidance.** The same-sample stack (9a) is a pattern, not an error - it is payable when the presumptive
result genuinely drove the confirmation, so treat volume and RATE, not instances, as the
signal: a provider stacking on 95% of samples is asserting that essentially every screen was
ambiguous. :stack_rate_cut = 0.50 is the starting cut: pairs are listed only for billers who
stack on more than half of their samples, so a biller with an occasional stack does not appear
at all; lower it to widen the list. :top_tier_share_cut = 0.80 for 9b reflects that G0483 (22+
drug classes) should be the exception, not the default. :min_tests = 25 is the volume floor for
both the stacking rate and the 9b share. Note that certain treatment populations legitimately
justify broad panels - exclude opioid treatment programs before drawing conclusions. Tune to
your population; a starting point, not a standard.

**5 · Paired control + failure mode.** CB-16 (Unbundling detection analytics, including the same-sample presumptive + definitive UDT code-pair edit). Secondary: CB-04 (NCCI PTP edit enforcement), CD-16 (Enhanced documentation review for high-cost / high-risk tests).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No unbundling analytics exist. CB-16's unbundling pattern alert does not exist. |
| **Designed-not-operating** | CB-16 is specified monthly; no alert was produced for the period. |
| **Operated-but-bypassed** | Alerts fired and no prepayment edit (CB-17) or provider education (PI-12) followed. |
| **Operated-but-ineffective** | The analytic detects fragmented surgical billing generically but carries no UDT-specific code-pair rule, which is the pattern CB-16 was written to catch. |
| **Evidence-missing** | Alerts exist without retained dispositions. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 10. Duplicate professional services across NPIs within one group

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
| **Secondary controls** | CB-09 - Payment-modifier validation (co-surgery 62, assistant 80/AS, …; CD-17 - Coordination-of-benefits / setting documentation control to prevent … |

**1 · Data hypothesis.** Duplicate detection that keys on (beneficiary, date, code, rendering NPI) will not catch the
same service billed twice by two providers in the same group, or by the group and by an
individual. The duplicate is real; the key is wrong. This is the coding-family analogue of the
cross-agency duplicate in the home-health pack, and OIG A-06-21-05003 (cited by CB-15) is the
reference finding.

**2 · What to check.** Group paid professional lines by beneficiary + date + code and flag groups containing more than
one line from different rendering NPIs sharing a group TIN, excluding codes where multiple
same-day services are expected.

**3 · Reference SQL.**

```sql
-- TEST 10 - Same service, same day, same group, different rendering NPI
-- Params: :period_start, :period_end

WITH candidate AS (
  SELECT
      cl.beneficiary_id, cl.service_date, cl.cpt, pr.group_tin,
      COUNT(*)                          AS line_count,
      COUNT(DISTINCT cl.rendering_npi)  AS distinct_renderers,
      COUNT(DISTINCT cl.billing_npi)    AS distinct_billers,
      SUM(cl.paid_amount)               AS combined_paid,
      MIN(cl.claim_line_id)             AS first_line,
      MAX(cl.claim_line_id)             AS last_line
  FROM claim_lines cl
  JOIN provider pr ON pr.npi = cl.rendering_npi
  WHERE cl.service_date BETWEEN :period_start AND :period_end
    AND cl.adjudication_status = 'PAID'
    AND cl.claim_type = 'PROFESSIONAL'
    -- exclude codes where repeat same-day services are expected
    AND cl.cpt NOT IN (SELECT hcpcs FROM repeat_service_exception_list)
    -- exclude lines already carrying a repeat modifier, which asserts the duplication
    -- (all four modifier slots are read)
    AND COALESCE(cl.modifier_1,'') NOT IN ('76','77','91')
    AND COALESCE(cl.modifier_2,'') NOT IN ('76','77','91')
    AND COALESCE(cl.modifier_3,'') NOT IN ('76','77','91')
    AND COALESCE(cl.modifier_4,'') NOT IN ('76','77','91')
  GROUP BY cl.beneficiary_id, cl.service_date, cl.cpt, pr.group_tin
)
SELECT *,
       CASE WHEN distinct_billers > 1 THEN 'CROSS_ENTITY' ELSE 'INTRA_GROUP' END AS duplicate_class
FROM candidate
WHERE distinct_renderers > 1
ORDER BY combined_paid DESC;
```

**4 · Threshold guidance.** Categorical once the exception list is right - and the exception list is the whole test. Build
`repeat_service_exception_list` before running: repeat labs, serial radiographs, anaesthesia,
critical-care add-ons and therapy units all legitimately repeat within a day. Without it this
query returns a large fraction of your professional claims and is useless. Modifiers 76, 77 and
91 explicitly assert a repeat service and are excluded above; if your population uses them
heavily, sample those separately rather than excluding them silently. Tune to your population;
a starting point, not a standard.

**5 · Paired control + failure mode.** CB-15 (Duplicate claim / line detection). Secondary: CB-09 (Payment-modifier validation), CD-17 (Coordination-of-benefits / setting documentation control).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | Duplicate detection covers exact claim resubmission only. |
| **Designed-not-operating** | CB-15's duplicate-hit disposition log is empty for the period. |
| **Operated-but-bypassed** | Duplicate hits were overridden at adjudication to clear a backlog. |
| **Operated-but-ineffective** | The duplicate key includes rendering_npi, so intra-group duplicates can never match. This is the defining defect for this test. |
| **Evidence-missing** | Hits suppressed silently in the edit engine with no retained log. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 11. Retired, deleted or not-yet-effective code paid

| | |
|---|---|
| **Paired control** | **CB-02** - Annual code-set update governance (HCPCS/CPT/ICD-10) |
| **Control type** | Preventive |
| **Module** | Module 03 - Coding & Billing Integrity |
| **Owner** | Coding compliance lead |
| **Frequency** | Annual |
| **System touchpoint** | Encoder / chargemaster |
| **Evidence artifact** | Code-set update change log |
| **Control failure mode** | Billing with retired/invalid codes |
| **Regulatory anchor** | 45 CFR 162.1002 (HIPAA code set standards) [binding] |
| **Secondary controls** | CB-18 - Pre-submission claim scrubber (provider side); CB-17 - Prepayment edit maintenance & tuning from emerging vulnerabilities |

**1 · Data hypothesis.** Code sets change annually (HCPCS/CPT effective 1 January; ICD-10 effective 1 October, with the
2027 set released 1 August 2026). Where code-set update governance is absent or the
chargemaster is refreshed late, claims pay against codes retired before the service date or not
yet effective on it. This is rarely fraud and almost always a control finding - which makes it
a clean, low-abrasion first test to run.

**2 · What to check.** Join each paid line's code to the code-set reference table and flag any service date outside
the code's effective/termination window. Roll up by billing entity and by code to distinguish a
systemic chargemaster problem from a single provider.

**3 · Reference SQL.**

```sql
-- TEST 11 - Service dates outside the code's effective window
-- Params: :period_start, :period_end

-- The code set is versioned by date: a code can have several rows (retired, then re-effective).
-- A line is clean when ANY version is in force on its service date; only a line with no version
-- in force is a defect, and it is classified against the code's whole history.
WITH code_range AS (
  SELECT code,
         MIN(effective_date) AS first_effective_date,
         MAX(COALESCE(termination_date, DATE '9999-12-31')) AS last_termination_date
  FROM code_set_reference
  GROUP BY code
)
SELECT
    cl.claim_line_id, cl.billing_npi, cl.rendering_npi, cl.beneficiary_id,
    cl.cpt AS billed_code, cl.service_date, cl.paid_amount,
    cr.first_effective_date, cr.last_termination_date,
    CASE
      WHEN cr.code IS NULL                                THEN 'CODE_NOT_IN_CODE_SET'
      WHEN cl.service_date < cr.first_effective_date      THEN 'NOT_YET_EFFECTIVE'
      ELSE 'RETIRED_CODE'
    END AS code_defect
FROM claim_lines cl
LEFT JOIN code_range cr ON cr.code = cl.cpt
WHERE cl.service_date BETWEEN :period_start AND :period_end
  AND cl.adjudication_status = 'PAID'
  AND NOT EXISTS (                                  -- no version in force on the service date
    SELECT 1 FROM code_set_reference cs
    WHERE cs.code = cl.cpt
      AND cl.service_date BETWEEN cs.effective_date
                              AND COALESCE(cs.termination_date, DATE '9999-12-31')
  )
ORDER BY code_defect, cl.paid_amount DESC;

-- Roll up: SELECT billing_npi, billed_code, code_defect, COUNT(*), SUM(paid_amount)
-- A single billing entity concentrated on one retired code is a chargemaster problem
-- (CB-02). Many entities on the same code is an EDIT problem (CB-17).
```

**4 · Threshold guidance.** Categorical, with one important caveat: your code-set reference table must be versioned by
date, not a current-state snapshot. A current-state table will report every claim for a code
retired last year as a defect, including the claims that were correct when submitted. Verify
that first - the count will be obviously implausible if you have not. The roll-up direction
(one entity vs many) is what tells you which control to fix, and that distinction is the point
of the test. Tune to your population; a starting point, not a standard.

**5 · Paired control + failure mode.** CB-02 (Annual code-set update governance - HCPCS/CPT/ICD-10). Secondary: CB-18 (Pre-submission claim scrubber), CB-17 (Prepayment edit maintenance & tuning).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No code-set update governance. CB-02's code-set update change log does not exist. |
| **Designed-not-operating** | CB-02 is annual; the change log shows no entry for the most recent release. |
| **Operated-but-bypassed** | The update was performed but exceptions were granted to keep legacy codes active in the chargemaster. |
| **Operated-but-ineffective** | The encoder was updated and the chargemaster was not, or vice versa - so one path still emits retired codes. |
| **Evidence-missing** | The update happened but no dated change log records what changed and when. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 12. Policy-required diagnosis absent from the claim

| | |
|---|---|
| **Paired control** | **CB-12** - Required-diagnosis-code edit (policy-mandated dx, e.g., OTP OUD) |
| **Control type** | Preventive |
| **Module** | Module 03 - Coding & Billing Integrity |
| **Owner** | Payer edit builder |
| **Frequency** | Per claim |
| **System touchpoint** | Prepayment edit engine |
| **Evidence artifact** | Required-dx edit specification |
| **Control failure mode** | Payment absent policy-required diagnosis |
| **Regulatory anchor** | OIG A-09-22-03005 (17 Aug 2023) - OTP opioid-use-disorder dx; all six recommendations closed [case evidence] |
| **Secondary controls** | CB-11 - Diagnosis-to-service medical-necessity linkage edit (NCD/LCD); PI-04 - Prepayment medical-review edits on high-risk codes/services |

**1 · Data hypothesis.** Some services are payable only with a specific diagnosis - the OIG opioid treatment program
finding cited by CB-12 is the reference case, where OTP bundles paid without an opioid-use-
disorder diagnosis. Where the required-diagnosis edit is absent or not specified for a service
class, these pay silently.

**2 · What to check.** For each service class carrying a policy-mandated diagnosis requirement, check whether any
diagnosis on the claim (or on a related claim in the lookback) satisfies it, and flag those
that do not.

**3 · Reference SQL.**

```sql
-- TEST 12 - Services paid without their policy-required diagnosis
-- Drive from a reference table of (service class -> acceptable ICD-10 set), maintained
-- alongside CB-12's required-dx edit specification.
-- Params: :period_start, :period_end, :dx_lookback_days (365)

SELECT
    cl.claim_line_id, cl.beneficiary_id, cl.billing_npi, cl.rendering_npi,
    cl.cpt, cl.service_date, cl.paid_amount,
    r.service_class, r.requirement_source,
    cd.claim_diagnoses
FROM claim_lines cl
JOIN required_dx_rule r
  ON  r.hcpcs = cl.cpt
  AND cl.service_date BETWEEN r.effective_date
                          AND COALESCE(r.termination_date, DATE '9999-12-31')
LEFT JOIN claim_diagnosis cd ON cd.claim_id = cl.claim_id
WHERE cl.service_date BETWEEN :period_start AND :period_end
  AND cl.adjudication_status = 'PAID'
  AND NOT EXISTS (
    -- required dx on THIS claim
    SELECT 1 FROM claim_diagnosis d
    JOIN required_dx_code rc ON rc.rule_id = r.rule_id AND rc.icd10_code = d.icd10_code
    WHERE d.claim_id = cl.claim_id
  )
  AND NOT EXISTS (
    -- or on any claim for this beneficiary in the lookback (a softer, fairer test)
    SELECT 1
    FROM claim_diagnosis d2
    JOIN claim_lines c2       ON c2.claim_id = d2.claim_id
    JOIN required_dx_code rc2 ON rc2.rule_id = r.rule_id AND rc2.icd10_code = d2.icd10_code
    WHERE c2.beneficiary_id = cl.beneficiary_id
      AND c2.service_date BETWEEN cl.service_date - :dx_lookback_days AND cl.service_date
  )
ORDER BY cl.paid_amount DESC;
```

**4 · Threshold guidance.** The requirement itself is categorical; the lookback is the judgement call. :dx_lookback_days =
365 is a fair starting window - requiring the diagnosis on the same claim is technically
correct but produces findings that are really claim-completion errors rather than coverage
failures. Run both: same-claim-only gives you the edit-configuration finding, lookback-
inclusive gives you the coverage finding. They point at different controls (CB-12 vs CB-11) and
should not be merged. Tune to your population; a starting point, not a standard.

**5 · Paired control + failure mode.** CB-12 (Required-diagnosis-code edit for policy-mandated diagnoses, e.g. OTP OUD). Secondary: CB-11 (Diagnosis-to-service medical-necessity linkage edit), PI-04 (Prepayment medical-review edits on high-risk codes/services).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No required-dx edit exists for the service class. CB-12's required-dx edit specification does not exist. |
| **Designed-not-operating** | The specification exists but the edit is not deployed in the adjudication path. |
| **Operated-but-bypassed** | The edit fires and is routinely overridden at adjudication. |
| **Operated-but-ineffective** | The edit checks the primary diagnosis position only, so a valid diagnosis in a secondary position fails and an invalid claim with the code in position one passes. |
| **Evidence-missing** | No retained record of edit hits and their disposition. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 13. Diagnosis-to-service medical-necessity linkage failure (NCD/LCD)

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
| **Secondary controls** | CD-01 - Coverage-criteria application: identify & apply the current …; CD-02 - Coverage-policy change monitoring: track the Medicare Coverage … |

**1 · Data hypothesis.** National and local coverage determinations define which diagnoses support which services. Where
the necessity-linkage edit is absent, or built once and never refreshed as coverage policy
changes, services pay without a supporting diagnosis - the largest single category of
insufficient-documentation improper payment.

**2 · What to check.** For services governed by an NCD or LCD with a covered-diagnosis list, check whether any
diagnosis on the claim appears on that list for the applicable jurisdiction and effective
period.

**3 · Reference SQL.**

```sql
-- TEST 13 - Services paid without an NCD/LCD-supported diagnosis
-- Coverage rules are jurisdiction-specific and dated. Load the Medicare Coverage
-- Database (or your own policy set) as a versioned reference; CB-11 and CD-02 own it.
-- Params: :period_start, :period_end

SELECT
    cl.claim_line_id, cl.beneficiary_id, cl.billing_npi, cl.rendering_npi,
    cl.cpt, cl.service_date, cl.paid_amount,
    pol.policy_id, pol.policy_type, pol.jurisdiction, pol.title
FROM claim_lines cl
JOIN coverage_policy pol
  ON  pol.hcpcs        = cl.cpt
  AND pol.jurisdiction = cl.mac_jurisdiction
  AND cl.service_date BETWEEN pol.effective_date
                          AND COALESCE(pol.retirement_date, DATE '9999-12-31')
WHERE cl.service_date BETWEEN :period_start AND :period_end
  AND cl.adjudication_status = 'PAID'
  AND pol.has_covered_dx_list = TRUE
  AND NOT EXISTS (
    SELECT 1
    FROM claim_diagnosis d
    JOIN coverage_policy_dx pdx
      ON pdx.policy_id = pol.policy_id AND pdx.icd10_code = d.icd10_code
    WHERE d.claim_id = cl.claim_id
  )
ORDER BY cl.paid_amount DESC;

-- Roll up by policy_id: a policy generating a very high failure rate across many
-- providers usually means the covered-dx list is loaded incorrectly or is out of date
-- - a CD-02 coverage-policy-monitoring finding, not a provider finding. Check that
-- before you open a single provider review.
```

**4 · Threshold guidance.** Categorical against the policy - but this test has the highest false-positive risk in the
family, because it depends entirely on the currency and completeness of your loaded coverage
policies. Run the by-policy roll-up first and set aside any policy with an implausibly high
failure rate; those are data problems. Restrict the first run to a handful of high-dollar,
well-understood policies rather than the whole coverage database. Tune to your population; a
starting point, not a standard.

**5 · Paired control + failure mode.** CB-11 (Diagnosis-to-service medical-necessity linkage edit - NCD/LCD). Secondary: CD-01 (Coverage-criteria application), CD-02 (Coverage-policy change monitoring).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No necessity-linkage edit exists. CB-11's necessity-edit disposition does not exist. |
| **Designed-not-operating** | The edit is configured but not active in the adjudication path for the period. |
| **Operated-but-bypassed** | Edit hits were overridden without a documented coverage rationale. |
| **Operated-but-ineffective** | The covered-diagnosis lists were loaded once and never refreshed against the Medicare Coverage Database, so the edit enforces retired policy - CD-02 is the upstream control. |
| **Evidence-missing** | No retained edit-disposition record. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 14. Assistant-at-surgery and co-surgery modifiers on ineligible procedures

| | |
|---|---|
| **Paired control** | **CB-09** - Payment-modifier validation (co-surgery 62, assistant 80/AS, bilateral 50) |
| **Control type** | Detective |
| **Module** | Module 03 - Coding & Billing Integrity |
| **Owner** | Coding auditor |
| **Frequency** | Per claim + sample |
| **System touchpoint** | Claim edit engine |
| **Evidence artifact** | Modifier-vs-documentation review |
| **Control failure mode** | Improper surgery/assistant payments (OIG) |
| **Regulatory anchor** | CMS Pub 100-04 Claims Processing Manual ch.12 sec.40.6-40.9 and 20.4.3 [guidance]; OIG A-01-20-00503 (22 Nov 2022) - co-surgery and assistant-at-surgery services billed without the appropriate modifiers [case evidence]; rec 23-A-01-018.04 open, .01 to .03 closed [case evidence] |
| **Secondary controls** | CB-04 - NCCI PTP edit enforcement pre-submission; CB-19 - Coding audit program (statistically valid sampling) + error-rate … |

**1 · Data hypothesis.** The physician fee schedule carries indicators governing whether an assistant surgeon, co-
surgeon or team surgeon is payable for each procedure. Where payment-modifier validation is
absent, modifiers 80/81/82/AS and 62/66 are appended to procedures where the indicator forbids
payment. OIG A-01-20-00503 (22 Nov 2022), cited by CB-09, is the reference finding.

**2 · What to check.** Join each line carrying an assistant, co-surgery or team-surgery modifier to the fee-schedule
indicator for that procedure and flag those where the indicator is restrictive. Report
separately the bilateral modifier 50 applied to procedures whose bilateral indicator forbids
it.

**3 · Reference SQL.**

```sql
-- TEST 14 - Surgical payment modifiers against fee-schedule indicators
-- Indicators: asst_surg 0/1/2/9, co_surg 0/1/2, team_surg 0/1/2, bilat_surg 0/1/2/3.
-- Restrictive values (0, and 9 for assistant) mean the modifier should not pay.
-- Params: :period_start, :period_end

WITH surgical AS (
  SELECT
      cl.claim_line_id, cl.beneficiary_id, cl.rendering_npi, cl.billing_npi,
      cl.cpt, cl.service_date, cl.paid_amount,
      cl.modifier_1, cl.modifier_2, cl.modifier_3, cl.modifier_4,
      f.asst_surg_indicator, f.co_surg_indicator,
      f.team_surg_indicator, f.bilat_surg_indicator,
      -- one flag per modifier family, read from all four modifier slots
      CASE WHEN COALESCE(cl.modifier_1,'') IN ('80','81','82','AS') OR COALESCE(cl.modifier_2,'') IN ('80','81','82','AS')
             OR COALESCE(cl.modifier_3,'') IN ('80','81','82','AS') OR COALESCE(cl.modifier_4,'') IN ('80','81','82','AS')
           THEN 1 ELSE 0 END AS has_assistant,
      CASE WHEN '62' IN (COALESCE(cl.modifier_1,''), COALESCE(cl.modifier_2,''), COALESCE(cl.modifier_3,''), COALESCE(cl.modifier_4,''))
           THEN 1 ELSE 0 END AS has_co_surgery,
      CASE WHEN '66' IN (COALESCE(cl.modifier_1,''), COALESCE(cl.modifier_2,''), COALESCE(cl.modifier_3,''), COALESCE(cl.modifier_4,''))
           THEN 1 ELSE 0 END AS has_team_surgery,
      CASE WHEN '50' IN (COALESCE(cl.modifier_1,''), COALESCE(cl.modifier_2,''), COALESCE(cl.modifier_3,''), COALESCE(cl.modifier_4,''))
           THEN 1 ELSE 0 END AS has_bilateral
  FROM claim_lines cl
  JOIN fee_schedule f
    ON  f.hcpcs = cl.cpt
    AND f.effective_year = EXTRACT(YEAR FROM cl.service_date)
  WHERE cl.service_date BETWEEN :period_start AND :period_end
    AND cl.adjudication_status = 'PAID'
),
classified AS (
  SELECT s.*,
         CASE
           WHEN s.has_assistant    = 1 AND s.asst_surg_indicator IN ('0','9')      THEN 'ASSISTANT_NOT_PAYABLE'
           WHEN s.has_co_surgery   = 1 AND s.co_surg_indicator = '0'               THEN 'CO_SURGERY_NOT_PAYABLE'
           WHEN s.has_team_surgery = 1 AND s.team_surg_indicator = '0'             THEN 'TEAM_SURGERY_NOT_PAYABLE'
           WHEN s.has_bilateral    = 1 AND s.bilat_surg_indicator IN ('0','2','3') THEN 'BILATERAL_NOT_APPLICABLE'
         END AS modifier_defect
  FROM surgical s
  WHERE s.has_assistant = 1 OR s.has_co_surgery = 1 OR s.has_team_surgery = 1 OR s.has_bilateral = 1
)
SELECT
    claim_line_id, beneficiary_id, rendering_npi, billing_npi,
    cpt, service_date, paid_amount,
    modifier_1, modifier_2, modifier_3, modifier_4,
    asst_surg_indicator, co_surg_indicator, team_surg_indicator, bilat_surg_indicator,
    modifier_defect
FROM classified
WHERE modifier_defect IS NOT NULL          -- defect list only: lines whose modifier is payable are not returned
ORDER BY modifier_defect, paid_amount DESC;

-- Note assistant indicator '2' means an assistant MAY be paid; '1' means documentation
-- must support it. Only 0 and 9 are flatly restrictive. Indicator '1' lines are a
-- SAMPLING population for CB-09's modifier-vs-documentation review, not a defect list,
-- and are not returned above; select them from `classified` with modifier_defect IS NULL
-- and the indicator = '1' when drawing that sample.
```

**4 · Threshold guidance.** Categorical against the indicator - but only for indicators 0 and 9. Indicator 1 (payable with
supporting documentation) is a sampling population, not a finding, and conflating the two is
the most common error in this test. Join the fee schedule on the year effective for the service
date; indicators change annually. Bilateral indicator 3 has different payment mechanics again  - 
verify how your adjudication handles it before treating it as a defect. Tune to your
population; a starting point, not a standard.

**5 · Paired control + failure mode.** CB-09 (Payment-modifier validation - co-surgery 62, assistant 80/AS, bilateral 50). Secondary: CB-04 (NCCI PTP edit enforcement), CB-19 (Coding audit program).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No payment-modifier validation exists. CB-09's modifier-vs-documentation review does not exist. |
| **Designed-not-operating** | CB-09 is per-claim plus sample; no sample was reviewed in the period. |
| **Operated-but-bypassed** | Restrictive-indicator hits were paid on override with no documentation review. |
| **Operated-but-ineffective** | The edit validates that the modifier is a valid code rather than that it is payable for the specific procedure. |
| **Evidence-missing** | No retained modifier-vs-documentation review workpaper. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 15. Telehealth modality - modifier, place of service and record disagree

| | |
|---|---|
| **Paired control** | **CD-18** - Service-modality documentation control: note and claim modifier support the billed modality (e.g., telehealth/audio-only) |
| **Control type** | Preventive |
| **Module** | Module 05 - Coverage & Documentation |
| **Owner** | Clinician / Billing |
| **Frequency** | Per service |
| **System touchpoint** | EHR / claim modifier |
| **Evidence artifact** | Modality-supporting note + modifier |
| **Control failure mode** | Billed modality not supported by the record (OIG psychotherapy telehealth) |
| **Regulatory anchor** | OIG A-09-21-03021 (2 May 2023) - psychotherapy incl. telehealth; all six recommendations closed [case evidence] |
| **Secondary controls** | CB-13 - Place-of-service / facility-vs-non-facility rate edit; CB-17 - Prepayment edit maintenance & tuning from emerging vulnerabilities |

**1 · Data hypothesis.** Telehealth and audio-only services are identified by place-of-service codes (02, 10) and
modifiers (95, 93, GT, FQ). Where the service-modality documentation control is absent, these
markers contradict each other - an in-person POS with a telehealth modifier, or a telehealth
POS on a service that cannot be furnished remotely. The OIG psychotherapy-telehealth finding
cited by CD-18 is the reference case.

**2 · What to check.** Cross-tabulate place of service against modality modifier for every service in the period and
flag the contradictory combinations, plus telehealth-flagged claims for codes absent from the
applicable telehealth services list.

**3 · Reference SQL.**

```sql
-- TEST 15 - Modality marker contradictions
-- Params: :period_start, :period_end

WITH tele AS (
  SELECT
      cl.claim_line_id, cl.beneficiary_id, cl.rendering_npi, cl.billing_npi,
      cl.cpt, cl.service_date, cl.place_of_service, cl.paid_amount,
      COALESCE(cl.modifier_1,'') AS mod1, COALESCE(cl.modifier_2,'') AS mod2,
      COALESCE(cl.modifier_3,'') AS mod3, COALESCE(cl.modifier_4,'') AS mod4,
      tl.hcpcs IS NOT NULL AS on_telehealth_list,
      -- COALESCE keeps a missing modifier from turning the test into NULL; all four slots read
      (COALESCE(cl.modifier_1,'') IN ('95','93','GT','FQ')
       OR COALESCE(cl.modifier_2,'') IN ('95','93','GT','FQ')
       OR COALESCE(cl.modifier_3,'') IN ('95','93','GT','FQ')
       OR COALESCE(cl.modifier_4,'') IN ('95','93','GT','FQ')) AS has_tele_modifier
  FROM claim_lines cl
  LEFT JOIN telehealth_service_list tl
         ON  tl.hcpcs = cl.cpt
         AND cl.service_date BETWEEN tl.effective_date
                                 AND COALESCE(tl.termination_date, DATE '9999-12-31')
  WHERE cl.service_date BETWEEN :period_start AND :period_end
    AND cl.adjudication_status = 'PAID'
),
classified AS (
  SELECT t.*,
    CASE
      WHEN t.place_of_service IN ('02','10') AND NOT t.has_tele_modifier
        THEN 'TELEHEALTH_POS_NO_MODIFIER'
      WHEN t.place_of_service NOT IN ('02','10') AND t.has_tele_modifier
        THEN 'MODIFIER_WITHOUT_TELEHEALTH_POS'
      WHEN t.place_of_service IN ('02','10') AND NOT t.on_telehealth_list
        THEN 'CODE_NOT_ON_TELEHEALTH_LIST'
    END AS modality_defect
  FROM tele t
)
SELECT
    claim_line_id, beneficiary_id, rendering_npi, billing_npi,
    cpt, service_date, place_of_service, paid_amount,
    mod1, mod2, mod3, mod4,
    on_telehealth_list, modality_defect
FROM classified
WHERE modality_defect IS NOT NULL                   -- only contradictions, not every telehealth claim
ORDER BY modality_defect, paid_amount DESC;
```

**4 · Threshold guidance.** Categorical, but heavily dependent on the effective-dated telehealth services list - which has
changed repeatedly since 2020 and continues to. Load it versioned by date or
CODE_NOT_ON_TELEHEALTH_LIST will be dominated by policy-vintage errors rather than billing
errors. Modifier positions matter: if your warehouse holds four modifier columns, extend the
checks past modifier_2 or you will miss any claim where the modality modifier landed in
position three. Tune to your population; a starting point, not a standard.

**5 · Paired control + failure mode.** CD-18 (Service-modality documentation control - note and claim modifier support the billed modality). Secondary: CB-13 (Place-of-service rate edit), CB-17 (Prepayment edit maintenance & tuning).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No modality documentation control exists. CD-18's modality-supporting note + modifier artifact does not exist. |
| **Designed-not-operating** | CD-18 is per-service; a population of paid telehealth services has no corresponding modality documentation. |
| **Operated-but-bypassed** | Contradictions were corrected on the claim without reviewing whether the record supports the modality actually furnished. |
| **Operated-but-ineffective** | The control checks that a modality modifier is present rather than that it agrees with the place of service and the record. |
| **Evidence-missing** | No retained evidence that modality was verified against the note. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 16. Edit-coverage blind spot - high-risk codes with zero prepayment edit activity

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
| **Secondary controls** | PI-04 - Prepayment medical-review edits on high-risk codes/services; CB-21 - High-risk-code monitoring dashboard to compliance/governance |

**1 · Data hypothesis.** This test inverts the others. Rather than looking for bad claims, it looks for controls that
are not working by looking for edits that never fire. Where prepayment edit maintenance is
absent, a code named in an OIG or GAO finding continues to pay at volume with no edit hit, no
denial and no review - the control's silence is the signal.

**2 · What to check.** For each code on your high-risk register (OIG/GAO findings, CERT/PERM error causes, internal
vulnerability list), compute paid volume, edit-hit count and denial rate in the period, and
flag codes with material spend and effectively zero control activity.

**3 · Reference SQL.**

```sql
-- TEST 16 - High-risk codes with material spend and no prepayment edit activity
-- Drive from your own high-risk register: the codes named in open OIG/GAO
-- recommendations, CERT/PERM error causes, and internal vulnerability findings.
-- Params: :period_start, :period_end, :min_paid (50000), :max_edit_hit_rate (0.001)

WITH paid AS (                              -- aggregate each side by code BEFORE joining,
  SELECT cl.cpt AS hcpcs,                   -- so neither side multiplies the other
         COUNT(cl.claim_line_id)       AS paid_lines,
         SUM(cl.paid_amount)           AS paid_amount,
         COUNT(DISTINCT cl.billing_npi) AS billing_entities
  FROM claim_lines cl
  WHERE cl.service_date BETWEEN :period_start AND :period_end
    AND cl.adjudication_status = 'PAID'
  GROUP BY cl.cpt
),
hits AS (
  SELECT eh.hcpcs, SUM(eh.hit_count) AS edit_hits
  FROM edit_hit_log eh
  WHERE eh.hit_date BETWEEN :period_start AND :period_end
  GROUP BY eh.hcpcs
),
activity AS (
  SELECT
      hr.hcpcs,
      hr.risk_source,                       -- e.g. 'OIG A-09-22-03010', 'CERT FY2025'
      hr.date_identified,
      COALESCE(p.paid_lines, 0)                                        AS paid_lines,
      COALESCE(p.paid_amount, 0)                                       AS paid_amount,
      COALESCE(p.billing_entities, 0)                                  AS billing_entities,
      COALESCE(h.edit_hits, 0)                                         AS edit_hits,
      COALESCE(h.edit_hits, 0) * 1.0 / NULLIF(p.paid_lines, 0)         AS edit_hit_rate   -- 1.0 avoids integer division
  FROM high_risk_code_register hr
  LEFT JOIN paid p ON p.hcpcs = hr.hcpcs
  LEFT JOIN hits h ON h.hcpcs = hr.hcpcs
)
SELECT
    hcpcs, risk_source, date_identified,
    paid_lines, paid_amount, billing_entities,
    edit_hits, ROUND(edit_hit_rate, 5) AS edit_hit_rate,
    CURRENT_DATE - date_identified AS days_since_risk_identified
FROM activity
WHERE paid_amount >= :min_paid
  AND edit_hit_rate <= :max_edit_hit_rate
ORDER BY paid_amount DESC;

-- This query's output is a CONTROL finding by construction, not a claim finding.
-- Do not open provider reviews from it; take it to the edit-change board (CB-17).
```

**4 · Threshold guidance.** :min_paid = 50,000 and :max_edit_hit_rate = 0.001 are arbitrary starting values and should be
set from your own spend distribution - the useful framing is 'material spend with no control
activity', and materiality is yours to define. The `days_since_risk_identified` column is the
one that matters in a governance conversation: a code named in an open OIG recommendation two
years ago, still paying at volume, with zero edit hits, is a complete and self-evident finding.
This test requires an edit-hit log; if you do not have one, that absence is itself the CB-17
finding. Tune to your population; a starting point, not a standard.

**5 · Paired control + failure mode.** CB-17 (Prepayment edit maintenance & tuning from emerging vulnerabilities). Secondary: PI-04 (Prepayment medical-review edits on high-risk codes/services), CB-21 (High-risk-code monitoring dashboard to compliance/governance).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No edit-change board and no high-risk register. CB-17's edit-change board record does not exist. |
| **Designed-not-operating** | CB-17 is quarterly; no edit-change board met or recorded a decision in the period. |
| **Operated-but-bypassed** | The board reviewed the vulnerability and deferred the edit indefinitely without a documented risk acceptance. |
| **Operated-but-ineffective** | Edits were added but configured so narrowly that they never fire - measurable precisely by this query's zero-hit result. |
| **Evidence-missing** | Edits were changed but no board record ties the change to the vulnerability that prompted it. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## Gap notes - controls this family needs that the library does not yet have

*Each note names a control objective these tests point to that the control library does not yet state. They are candidates for a future library release; the library itself is unchanged by this pack.*

### G1. No control names Modifier 25, the highest-volume edit-bypass modifier.

CB-05 is titled 'Modifier-59 / X{EPSU} governance & documentation review', but its trigger
reads 'Modifier appended to bypass an edit' - generic, and a fair home for modifier 25. The
mismatch matters: an assessor reading the control name will scope the audit sample to modifier
59 and X{EPSU} and never sample modifier 25, which is exactly the 'Operated-but-ineffective'
failure mode recorded against TEST 2. Modifier-25 overuse is one of the highest-signal tests in
this pack.

**Recommendation.** Known gap, with no control change made in this pack. Either (a) widen CB-05's name and scope to
'edit-bypass modifier governance' covering 25, 59, X{EPSU}, 24, 57, 76/77/91 with the sampling
requirement stated per modifier, or (b) author a dedicated Modifier-25 control in Module 03.
Option (a) is probably better - it future-proofs against the next bypass modifier - but it
changes an existing control's scope, which needs the parity re-assertion the module process
requires.

### G2. No control names the new-patient vs established-patient E&M rule.

TEST 4 is paired to CB-19 (the generic statistically-valid coding audit program) because
nothing more specific exists. The new-patient rule is a bright-line, high-dollar, entirely
data-testable requirement - the kind of thing the control library covers well elsewhere - and
its absence means an assessor has no prompt to test it.

**Recommendation.** Candidate control for Module 03: new-vs-established E&M status verification at coding, with the
three-year same-specialty-same-group lookback stated in the design and a scrubber edit (CB-18)
as the preventive counterpart.

### G3. Clinical validation for the highest-audited DRGs is absent.

CB-14 covers discharge-status and DRG coding accuracy, which is a coding-integrity question. It
does not cover clinical validation - whether the documented clinical picture supports the coded
diagnosis - for sepsis, encephalopathy and malnutrition, the three diagnoses MII runs a
dedicated boot camp on and the highest-audited DRGs in the market. No test in this family
addresses it, because no control does.

**Recommendation.** Known gap: the analytics gap tracks the control gap. Once Module 03/05 carry clinical-
validation controls, the paired analytics should be added (e.g. sepsis DRG coded without
supporting lactate/culture/vasopressor evidence in the claim record).

---

*Reference patterns. Public-source, personal-capacity, advisory and organizational only. Queries must be adapted to your own schema and tested on your own warehouse before use. Outputs are review candidates for human adjudication, never payment or denial dispositions. Confirm every regulatory anchor against its primary source.*