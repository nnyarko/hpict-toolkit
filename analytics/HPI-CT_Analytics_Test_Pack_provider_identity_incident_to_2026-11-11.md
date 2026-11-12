# Analytics Test Pack - Provider identity, credential integrity, and incident-to billing

*is the provider on the claim the provider who did the work - and do they exist*

*Generated 2026-11-11 from the HPI-CT control library (`data/controls.csv`, 173 controls). 16 detection tests, paired to 16 primary controls. Reference patterns, not programs.*

## Before you run anything

**These queries produce review candidates, never payment or denial decisions.** Every hit is a question for a human.

**The toolkit never receives your data.** These are patterns you run inside your own environment. No claim data, no PHI, no identifiers leave your systems.

**Every query is a pattern, not a program.** Field names are generic; yours will differ. Expect to rewrite the FROM and JOIN clauses and to remap service categories to your own taxonomy.

**Every threshold is a starting point, not a standard.** Where a threshold is arithmetic it is labelled as such. Everywhere else, measure your own distribution first.

Read `PREFACE_how_to_run_these.md` before the first run.

## Scope of this family

Everything that turns on the question of who rendered a service and whether that person or
entity is what the enrollment record says. Four related surfaces: **identity** (stolen, rented
and synthetic provider identities), **credential integrity** (services outside a provider's
scope, stale or lapsed enrollment), **supervision** (incident-to and split/shared billing,
where the NPI on the claim is deliberately not the person who did the work), and **entity
structure** (shell networks, undisclosed ownership, rapid acquisition). This family draws most
heavily on Module 01, which until now had almost no analytics coverage - three controls across
the first four families.

Provider-identity fraud is the category where the claim itself is perfectly formed. The service
may even have been delivered; the coding may be immaculate. What is wrong is the identity
attached to it, and no amount of claim-level edit logic will see that, because every field
validates. Detection here works by joining claims to something outside the claim - the
enrollment record, the ownership graph, the credential registry, the provider's own physical
footprint. That makes this family unusually dependent on reference-data quality, and unusually
rewarding when that data is good.

## Tests at a glance

| # | Test | Primary control | Also touches |
|---|---|---|---|
| 1 | Implausible daily service volume for one rendering NPI | **CB-08** | PI-20, CB-07 |
| 2 | Footprint inconsistency - billing from locations the enrollment record does not support | **PI-20** | PE-02, PE-11 |
| 3 | Incident-to and split/shared billing - supervising NPI with no physician footprint | **CD-06** | CD-03, PI-20 |
| 4 | Billing after revalidation lapse or deactivation | **PE-14** | PE-02, PE-09 |
| 5 | New-provider billing velocity across all provider types | **PE-15** | PE-16, PI-02 |
| 6 | Acquisition velocity - one owner accumulating billing numbers | **PE-16** | PE-05, PE-06 |
| 7 | Services outside the provider's credentialed scope or taxonomy | **PE-01** | PE-18, CB-01 |
| 8 | Enrollment-of-record staleness - claims contradicting the enrollment file | **PE-02** | PE-06, MD-16 |
| 9 | Ownership network - entities sharing owners, addresses, or banking details | **PE-05** | PE-04, MD-03 |
| 10 | Undisclosed ownership or control change within the reporting window | **PE-06** | MD-03, PE-04 |
| 11 | Multi-source link analysis - cross-provider scheme convergence | **PI-03** | PE-05, PI-22 |
| 12 | Credentialing re-attestation lapse in payer networks | **PE-18** | PE-01, MC-11 |
| 13 | Synthetic-identity indicators at enrollment | **PE-20** | PE-01, PE-13 |
| 14 | Downstream entity rendering services but absent from the FDR inventory | **MC-11** | PE-18, MD-10 |
| 15 | Entity-network cash loop - claim proceeds recycling among related parties | **PI-22** | PE-05, PI-23 |
| 16 | AI-fabricated supporting documentation - corroboration mismatch | **PI-24** | CD-20, CD-07 |

> **Control-ID validation:** every control referenced above resolves to a real ID in the current control library. See the gap notes at the end for controls this family *needs* but the library does not yet have.

---

## 1. Implausible daily service volume for one rendering NPI

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
| **Secondary controls** | PI-20 - Provider NPI/credential-footprint consistency monitoring - flag …; CB-07 - MUE units-of-service edit enforcement |

**1 · Data hypothesis.** One clinician can see a bounded number of patients in a day. Where quantity and frequency edits
operate per code rather than per provider-day, an NPI can carry a total daily workload no
individual could perform - the signature of a rented or stolen identity, of incident-to volume
rolled up under one physician, or simply of billing for encounters that did not occur. This is
the general-encounter counterpart to family 1's time-based impossible-day test.

**2 · What to check.** For each rendering NPI and date, count distinct beneficiaries and total encounters across all
service types, and convert to an implied clinical workload using typical service times. Flag
provider-days above a plausibility ceiling and, more usefully, providers with a recurring
pattern of them.

**3 · Reference SQL.**

```sql
-- TEST 1 - Implied daily workload per rendering NPI, all service types
-- Params: :period_start, :period_end, :implied_minutes_ceiling (720), :recurrence_threshold (5),
--         :default_service_minutes (20)

WITH daily AS (
  SELECT
      cl.rendering_npi,
      cl.service_date,
      COUNT(*)                          AS lines,
      COUNT(DISTINCT cl.beneficiary_id) AS beneficiaries,
      COUNT(DISTINCT cl.billing_npi)    AS billing_entities,
      COUNT(DISTINCT cl.place_of_service) AS distinct_pos,
      -- implied clinical minutes from a service-time reference table you maintain
      SUM(COALESCE(st.typical_minutes, :default_service_minutes) * cl.units) AS implied_minutes,
      SUM(cl.paid_amount) AS paid_amount
  FROM claim_lines cl
  LEFT JOIN service_time_reference st ON st.hcpcs = cl.hcpcs
  WHERE cl.service_date BETWEEN :period_start AND :period_end
    AND cl.adjudication_status = 'PAID'
    AND cl.rendering_npi IS NOT NULL
    AND cl.claim_type = 'PROFESSIONAL'
  GROUP BY cl.rendering_npi, cl.service_date
),
flagged AS (
  SELECT * FROM daily WHERE implied_minutes > :implied_minutes_ceiling
)
SELECT
    f.rendering_npi, pr.specialty, pr.peer_region,
    COUNT(*)                       AS implausible_days,
    ROUND(AVG(f.implied_minutes) / 60.0, 1) AS mean_implied_hours,
    MAX(f.beneficiaries)           AS max_beneficiaries_in_a_day,
    MAX(f.billing_entities)        AS max_billing_entities_in_a_day,
    SUM(f.paid_amount)             AS paid_on_implausible_days
FROM flagged f
LEFT JOIN provider pr ON pr.npi = f.rendering_npi
GROUP BY f.rendering_npi, pr.specialty, pr.peer_region
HAVING COUNT(*) >= :recurrence_threshold
ORDER BY mean_implied_hours DESC, paid_on_implausible_days DESC;

-- max_billing_entities_in_a_day is the identity signal hiding inside a volume test:
-- one rendering NPI appearing under several unrelated billing entities on the same day
-- is a rented-credential pattern, not a busy-clinician pattern.
```

**4 · Threshold guidance.** :implied_minutes_ceiling = 720 (12 clinical hours) is a deliberately generous starting ceiling,
and :recurrence_threshold = 5 days matters more - an isolated heavy day is normal, a recurring
impossible one is not. The service-time reference table is where the judgement sits, and it
should be built with clinical input rather than from fee-schedule RVU time, which measures
something different. Two specialties will dominate false positives unless handled: radiology
and pathology, where interpretive volume is legitimately very high and per-encounter time is
not meaningful. Exclude them or give them their own ceiling. Tune to your population; a
starting point, not a standard.

**5 · Paired control + failure mode.** CB-08 (Quantity / frequency-limit edits). Secondary: PI-20 (Provider NPI/credential-footprint consistency monitoring), CB-07 (MUE units-of-service edit enforcement).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No provider-day aggregate exists anywhere in the edit or analytics set. CB-08's frequency-limit exception report does not cover encounter volume. |
| **Designed-not-operating** | CB-08 is continuous; no exception report was produced for the period. |
| **Operated-but-bypassed** | Implausible-volume exceptions were closed without establishing who rendered the services. |
| **Operated-but-ineffective** | Edits operate per code and per claim, so a provider-day assembled from many different codes across many claims never exceeds any single limit. |
| **Evidence-missing** | Analysis exists in a BI tool with no dated exception log or reviewer disposition. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 2. Footprint inconsistency - billing from locations the enrollment record does not support

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
| **Secondary controls** | PE-02 - PECOS/enrollment-of-record data reconciliation cycle; PE-11 - On-site visits for Moderate/High-risk providers |

**1 · Data hypothesis.** PI-20 exists precisely for this: a legitimately enrolled NPI billed by an unrelated party for
services the named provider never rendered. Where footprint monitoring is absent, claims appear
from service locations, geographies or billing entities inconsistent with the provider's
enrollment of record and historical pattern - the defining signature of a stolen or rented
credential.

**2 · What to check.** Join paid claims to the enrollment record and to the provider's own historical claim geography,
and flag claims from locations outside both. Geography is compared by state and 3-digit ZIP;
read the result by how many new locations appeared and how abruptly.

**3 · Reference SQL.**

```sql
-- TEST 2 - Claims from locations inconsistent with enrollment and history
-- Geography is compared by state and 3-digit ZIP only. There is no distance measure here:
-- a move to the adjoining ZIP3 and a move across the state are both NEW_ZIP3_NO_HISTORY.
-- If you hold coordinates (or a ZIP3 centroid table), add the distance as a ranking column.
-- Params: :period_start, :period_end, :baseline_start
-- Portability: date +/- integer days and date - date (a day count) are PostgreSQL/DuckDB style;
--   elsewhere use DATEADD / DATE_ADD for the offset and DATEDIFF / DATE_DIFF for the difference.

WITH baseline_geo AS (
  -- where this NPI historically billed from, before the period under test
  SELECT cl.rendering_npi,
         loc.state_code,
         loc.zip3,
         COUNT(*) AS baseline_lines
  FROM claim_lines cl
  JOIN service_location loc ON loc.location_id = cl.service_location_id
  WHERE cl.service_date BETWEEN :baseline_start AND :period_start - 1
    AND cl.adjudication_status = 'PAID'
  GROUP BY cl.rendering_npi, loc.state_code, loc.zip3
),
enrolled_geo AS (
  SELECT e.npi, e.practice_state, e.practice_zip3
  FROM provider_enrollment_location e
  WHERE e.status = 'ACTIVE'
)
SELECT
    cl.claim_line_id, cl.rendering_npi, cl.billing_npi,
    cl.service_date, cl.hcpcs, cl.paid_amount,
    loc.state_code AS claim_state, loc.zip3 AS claim_zip3,
    eg.practice_state, eg.practice_zip3,
    bg.baseline_lines,
    CASE
      WHEN eg.npi IS NULL                       THEN 'NO_ACTIVE_ENROLLMENT_LOCATION'
      WHEN loc.state_code <> eg.practice_state
       AND bg.rendering_npi IS NULL             THEN 'NEW_STATE_NOT_ENROLLED_NOT_HISTORICAL'
      ELSE 'NEW_ZIP3_NO_HISTORY'                -- enrolled, but no history at this state + ZIP3
    END AS footprint_class
FROM claim_lines cl
JOIN service_location loc ON loc.location_id = cl.service_location_id
LEFT JOIN enrolled_geo eg ON eg.npi = cl.rendering_npi
LEFT JOIN baseline_geo bg ON bg.rendering_npi = cl.rendering_npi
                         AND bg.state_code    = loc.state_code
                         AND bg.zip3          = loc.zip3
WHERE cl.service_date BETWEEN :period_start AND :period_end
  AND cl.adjudication_status = 'PAID'
  AND (eg.npi IS NULL OR bg.rendering_npi IS NULL)
ORDER BY footprint_class, cl.paid_amount DESC;

-- Roll up by rendering_npi. A provider with ONE new location has moved or added a site.
-- A provider appearing in several unrelated new geographies in the same period, under
-- billing entities they have no history with, is the pattern PI-20 was written for.
```

**4 · Threshold guidance.** The classification is categorical but the interpretation is not. Locum tenens, telehealth
across state lines, multi-site group practices, hospital coverage arrangements and simple
relocation all produce new geography lawfully, and any of them will generate volume here. The
discriminating factors are multiplicity and abruptness: several unrelated new geographies
appearing at once, under billing entities the provider has no history with, while the
provider's original-location volume continues unchanged. Build the roll-up before reading the
line detail, or you will drown in correct claims. Tune to your population; a starting point,
not a standard.

**5 · Paired control + failure mode.** PI-20 (Provider NPI/credential-footprint consistency monitoring). Secondary: PE-02 (PECOS/enrollment-of-record data reconciliation), PE-11 (Unannounced site visits for Moderate/High-risk providers).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No claims-to-enrollment footprint join exists. PI-20's footprint-inconsistency alert log does not exist. |
| **Designed-not-operating** | PI-20 is specified continuous / near-real-time; no alerts were produced in the period. |
| **Operated-but-bypassed** | Alerts were cleared as 'provider added a location' with no confirmation against the enrollment record. |
| **Operated-but-ineffective** | The monitor compares each claim to the enrollment address only and has no historical-pattern baseline, so it cannot distinguish a new legitimate site from an abrupt anomalous one. |
| **Evidence-missing** | Alerts fire into a queue with no retained dispositions. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 3. Incident-to and split/shared billing - supervising NPI with no physician footprint

| | |
|---|---|
| **Paired control** | **CD-06** - Signature legibility & attestation control: legible identifiable author signature on each entry; attestation for missing/illegible signatures |
| **Control type** | Preventive |
| **Module** | Module 05 - Coverage & Documentation |
| **Owner** | HIM |
| **Frequency** | Per record |
| **System touchpoint** | EHR signature log |
| **Evidence artifact** | Signature or signature-attestation |
| **Control failure mode** | Unsigned/illegible records denied on review (PIM 100-08 Ch.3) |
| **Regulatory anchor** | Medicare Program Integrity Manual (CMS Pub 100-08) Ch.3 [guidance] |
| **Secondary controls** | CD-03 - Reasonable-and-necessary documentation control: the medical record …; PI-20 - Provider NPI/credential-footprint consistency monitoring - flag … |

**1 · Data hypothesis.** Incident-to billing lets services furnished by auxiliary personnel be billed under a
supervising physician's NPI at the physician rate, conditional on direct supervision and an
established plan of care. Where no control tests the supervision condition, a physician's NPI
carries volume on days the physician was demonstrably elsewhere - or carries volume no single
physician could supervise. The service was probably delivered; the payment rate and the
identity are the issue.

**2 · What to check.** For each physician NPI, identify days where incident-to-eligible services were billed under
their NPI and test for any evidence of the physician's own presence: a personally-performed
service, a signed note, or a service at that location. Then test supervisory span - how many
auxiliary personnel one NPI implies.

**3 · Reference SQL.**

```sql
-- TEST 3 - Incident-to volume with no supervising-physician footprint
-- Incident-to and split/shared rules differ by setting and change; confirm current
-- requirements before drawing conclusions. This test infers presence from claims.
-- Params: :period_start, :period_end

WITH physician_days AS (
  SELECT
      cl.rendering_npi AS supervising_npi,
      cl.service_date,
      cl.service_location_id,
      COUNT(*) AS incident_to_eligible_lines,
      COUNT(DISTINCT cl.beneficiary_id) AS beneficiaries,
      SUM(cl.paid_amount) AS paid_amount
  FROM claim_lines cl
  JOIN provider pr ON pr.npi = cl.rendering_npi
  WHERE cl.service_date BETWEEN :period_start AND :period_end
    AND cl.adjudication_status = 'PAID'
    AND pr.provider_class = 'PHYSICIAN'
    AND cl.place_of_service IN ('11','49')          -- office settings; incident-to scope
    AND cl.hcpcs IN (SELECT hcpcs FROM incident_to_eligible_code_list)
  GROUP BY cl.rendering_npi, cl.service_date, cl.service_location_id
)
SELECT
    pd.supervising_npi, pd.service_date, pd.service_location_id,
    pd.incident_to_eligible_lines, pd.beneficiaries, pd.paid_amount,
    -- evidence of the physician personally performing anything that day
    (SELECT COUNT(*) FROM claim_lines p
      WHERE p.rendering_npi = pd.supervising_npi
        AND p.service_date  = pd.service_date
        AND p.service_location_id = pd.service_location_id   -- same site only
        AND p.adjudication_status = 'PAID'
        AND p.hcpcs NOT IN (SELECT hcpcs FROM incident_to_eligible_code_list))
      AS personally_performed_lines,
    -- physician billing at a DIFFERENT location the same day
    (SELECT COUNT(DISTINCT o.service_location_id) FROM claim_lines o
      WHERE o.rendering_npi = pd.supervising_npi
        AND o.service_date  = pd.service_date
        AND o.service_location_id <> pd.service_location_id
        AND o.adjudication_status = 'PAID')
      AS other_locations_same_day,
    -- signed-note evidence, where you hold EHR authorship data
    (SELECT COUNT(*) FROM ehr_signature s
      WHERE s.author_npi = pd.supervising_npi
        AND CAST(s.signed_at AS DATE) = pd.service_date)
      AS signed_notes_that_day
FROM physician_days pd
WHERE NOT EXISTS (
      SELECT 1 FROM claim_lines p
      WHERE p.rendering_npi = pd.supervising_npi
        AND p.service_date  = pd.service_date
        AND p.service_location_id = pd.service_location_id   -- same site only
        AND p.adjudication_status = 'PAID'
        AND p.hcpcs NOT IN (SELECT hcpcs FROM incident_to_eligible_code_list))
ORDER BY pd.paid_amount DESC;
```

**4 · Threshold guidance.** There is no threshold here, only a chain of inference, and it should be stated honestly:
absence of a personally-performed claim on a given day is weak evidence of absence from the
office. A physician can supervise all day and personally bill nothing. What strengthens the
inference is `other_locations_same_day` greater than zero - the physician was demonstrably
somewhere else - and volume that implies an impossible supervisory span. Rank on those two,
treat the rest as context, and note that the remedy for a confirmed finding is usually a rate
adjustment rather than a denial, since the service was furnished. Tune to your population; a
starting point, not a standard.

**5 · Paired control + failure mode.** CD-06 (Signature legibility & attestation control - legible identifiable author on each entry). Secondary: CD-03 (Reasonable-and-necessary documentation), PI-20 (Provider NPI/credential-footprint consistency monitoring). See gap note G1 - no control addresses supervision or incident-to conditions.

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No control tests supervision conditions at all. CD-06 covers authorship of the record, not the supervisory relationship behind the billing identity. |
| **Designed-not-operating** | CD-06 is per-record; a population of incident-to claims has no corresponding attributable signature. |
| **Operated-but-bypassed** | Notes were signed by the supervising physician as a batch attestation with no involvement in the encounters. |
| **Operated-but-ineffective** | The control confirms a signature exists and is legible but never tests whether the signing physician met the supervision condition - this is the whole gap. |
| **Evidence-missing** | No retained signature log linking encounters to a supervising physician. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 4. Billing after revalidation lapse or deactivation

| | |
|---|---|
| **Paired control** | **PE-14** - Revalidation cycle enforcement + auto-deactivation |
| **Control type** | Detective |
| **Module** | Module 01 - Provider Enrollment |
| **Owner** | Enrollment ops |
| **Frequency** | Per cycle |
| **System touchpoint** | PECOS revalidation queue |
| **Evidence artifact** | Revalidation status report |
| **Control failure mode** | Dormant/abandoned billing numbers misused |
| **Regulatory anchor** | 42 CFR 424 Subpart P [binding]; CMS PIM Pub 100-08 Ch.10 [guidance] |
| **Secondary controls** | PE-02 - PECOS/enrollment-of-record data reconciliation cycle; PE-09 - State Medicaid exclusion / NPDB / death-master checks |

**1 · Data hypothesis.** Revalidation exists to confirm an enrolled provider still is what they claimed to be, and
deactivation exists to close dormant billing numbers before someone else uses them. Where the
revalidation cycle is not enforced, providers bill past their revalidation due date, and
dormant numbers reactivate. PE-14's own stated failure mode is 'dormant/abandoned billing
numbers misused' - this test measures it.

**2 · What to check.** Flag paid claims with a service date after the revalidation due date and before any completed
revalidation, and separately identify NPIs that billed nothing for an extended period and then
resumed at volume - the dormancy-reactivation pattern.

**3 · Reference SQL.**

```sql
-- TEST 4 - Claims after revalidation lapse, and dormant-NPI reactivation
-- 42 CFR 424 Subpart P; 5-year cycle, 3-year for DMEPOS.
-- 4b compares paid amounts in two windows of :comparison_days: the days up to the last claim
-- before the gap, and the days from the first claim after it. A resumption paying at least
-- :reactivation_multiple times the pre-gap window is labelled DORMANT_REACTIVATION_AT_VOLUME;
-- any other resumption after the gap stays DORMANT_REACTIVATION.
-- Params: :period_start, :period_end, :dormancy_days (365), :reactivation_multiple (3.0),
--         :comparison_days (90)
-- Portability: date +/- integer days and date - date (a day count) are PostgreSQL/DuckDB style;
--   elsewhere use DATEADD / DATE_ADD for the offset and DATEDIFF / DATE_DIFF for the difference.

-- 4a: billed past the revalidation due date
SELECT
    'REVALIDATION_LAPSE' AS finding,
    cl.claim_line_id, cl.billing_npi, cl.rendering_npi,
    cl.service_date, cl.paid_amount,
    e.revalidation_due_date, e.last_revalidation_date, e.enrollment_status,
    (cl.service_date - e.revalidation_due_date) AS days_past_due,
    CAST(NULL AS DECIMAL(18,2)) AS resume_to_pre_gap_ratio
FROM claim_lines cl
JOIN provider_enrollment e ON e.npi = cl.billing_npi
WHERE cl.service_date BETWEEN :period_start AND :period_end
  AND cl.adjudication_status = 'PAID'
  AND e.revalidation_due_date < cl.service_date
  AND (e.last_revalidation_date IS NULL OR e.last_revalidation_date < e.revalidation_due_date)

UNION ALL

-- 4b: dormant NPI resuming at volume
SELECT
    CASE WHEN d.paid_resume_window >= :reactivation_multiple * d.paid_pre_gap_window
         THEN 'DORMANT_REACTIVATION_AT_VOLUME'
         ELSE 'DORMANT_REACTIVATION' END AS finding,
    CAST(NULL AS VARCHAR) AS claim_line_id,
    d.billing_npi, CAST(NULL AS VARCHAR) AS rendering_npi,
    d.resume_date AS service_date, d.paid_after_resume AS paid_amount,
    CAST(NULL AS DATE), d.last_claim_before_gap, CAST(NULL AS VARCHAR),
    d.gap_days AS days_past_due,
    ROUND(d.paid_resume_window / NULLIF(d.paid_pre_gap_window, 0), 2) AS resume_to_pre_gap_ratio
FROM (
    SELECT
        g.billing_npi, g.last_claim_before_gap, g.resume_date,
        g.resume_date - g.last_claim_before_gap AS gap_days,
        SUM(CASE WHEN cl.service_date >= g.resume_date THEN cl.paid_amount ELSE 0 END)
          AS paid_after_resume,
        -- the :comparison_days up to and including the last claim before the gap
        SUM(CASE WHEN cl.service_date >  g.last_claim_before_gap - :comparison_days
                  AND cl.service_date <= g.last_claim_before_gap
                 THEN cl.paid_amount ELSE 0 END) AS paid_pre_gap_window,
        -- the :comparison_days starting at the first claim after the gap
        SUM(CASE WHEN cl.service_date >= g.resume_date
                  AND cl.service_date <  g.resume_date + :comparison_days
                 THEN cl.paid_amount ELSE 0 END) AS paid_resume_window
    FROM (
        SELECT
            c0.billing_npi,
            MAX(CASE WHEN c0.service_date <  :period_start THEN c0.service_date END) AS last_claim_before_gap,
            MIN(CASE WHEN c0.service_date >= :period_start THEN c0.service_date END) AS resume_date
        FROM claim_lines c0
        WHERE c0.adjudication_status = 'PAID'
        GROUP BY c0.billing_npi
    ) g
    JOIN claim_lines cl ON cl.billing_npi = g.billing_npi AND cl.adjudication_status = 'PAID'
    GROUP BY g.billing_npi, g.last_claim_before_gap, g.resume_date
) d
WHERE d.gap_days >= :dormancy_days
  AND d.paid_after_resume > 0
ORDER BY finding, paid_amount DESC;
```

**4 · Threshold guidance.** :dormancy_days = 365 with any resumed billing is a reasonable opening cut, but a long gap has
many innocent causes - a solo practitioner on medical or parental leave, a seasonal practice, a
provider who moved payers and came back. What makes reactivation interesting is the *shape* of
the resumption: volume immediately at or above the pre-gap level, a changed billing address,
changed banking details, or a service mix unlike anything the NPI billed before. The query
makes the volume comparison itself (:reactivation_multiple = 3.0 over :comparison_days = 90
either side of the gap) and labels the result; a resumption window cut short by the end of your
data understates the ratio. Add the address, banking and service-mix comparisons before acting.
Revalidation lapse (4a) is categorical and needs no tuning, but confirm how your enrollment
system records extensions and pending revalidations before treating a past-due date as a lapse.
Tune to your population; a starting point, not a standard.

**5 · Paired control + failure mode.** PE-14 (Revalidation cycle enforcement + auto-deactivation). Secondary: PE-02 (PECOS/enrollment-of-record data reconciliation), PE-09 (State Medicaid exclusion / NPDB / death-master checks).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No revalidation tracking. PE-14's revalidation status report does not exist. |
| **Designed-not-operating** | The cycle is defined but the queue was not worked, so due dates passed without action. |
| **Operated-but-bypassed** | Providers past due continued to be paid pending revalidation with no payment hold applied. |
| **Operated-but-ineffective** | Revalidation is tracked but deactivation is not automatic, so a lapse produces a report and no consequence - the 'auto-deactivation' half of the control is the part that does the work. |
| **Evidence-missing** | Revalidations occurred but no status report evidences which were completed on time. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 5. New-provider billing velocity across all provider types

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

**1 · Data hypothesis.** Bust-out billing depends on speed: enroll, bill hard, exit before any review cycle arrives.
Where the velocity watch is absent, an entity can reach peak billing within weeks and be gone
before its first revalidation. PE-15 cites Operation Gold Rush directly. This generalises
family 1's home-health-scoped version to every provider type.

**2 · What to check.** For providers enrolled within the lookback, compute cumulative paid amount, distinct
beneficiaries and beneficiary-per-day acquisition rate across the first window, and compare to
the same-type, same-region enrollment cohort.

**3 · Reference SQL.**

```sql
-- TEST 5 - First-window billing velocity by enrollment cohort
-- Params: :period_start, :period_end, :velocity_window_days (90), :percentile_cut (0.95),
--         :suspicious_first_claim_days (7)
-- Portability: date +/- integer days and date - date (a day count) are PostgreSQL/DuckDB style;
--   elsewhere use DATEADD / DATE_ADD for the offset and DATEDIFF / DATE_DIFF for the difference.

WITH cohort AS (
  SELECT e.npi AS billing_npi, e.enrollment_effective_date,
         e.provider_type, e.peer_region, e.risk_level
  FROM provider_enrollment e
  WHERE e.enrollment_effective_date BETWEEN :period_start AND :period_end
),
early AS (
  SELECT
      c.billing_npi, c.enrollment_effective_date, c.provider_type, c.peer_region, c.risk_level,
      SUM(cl.paid_amount)                AS paid_first_window,
      COUNT(DISTINCT cl.beneficiary_id)  AS beneficiaries_first_window,
      COUNT(DISTINCT cl.rendering_npi)   AS renderers_first_window,
      MIN(cl.service_date)               AS first_service_date,
      MIN(cl.service_date) - c.enrollment_effective_date AS days_to_first_claim,
      COUNT(DISTINCT cl.service_date)    AS active_days
  FROM cohort c
  JOIN claim_lines cl
    ON  cl.billing_npi   = c.billing_npi
    AND cl.service_date >= c.enrollment_effective_date
    AND cl.service_date <  c.enrollment_effective_date + :velocity_window_days
  WHERE cl.adjudication_status = 'PAID'
  GROUP BY c.billing_npi, c.enrollment_effective_date, c.provider_type,
           c.peer_region, c.risk_level
)
SELECT
    e.*,
    ROUND(e.beneficiaries_first_window * 1.0 / NULLIF(e.active_days, 0), 2)
      AS beneficiaries_per_active_day,
    ROUND(PERCENT_RANK() OVER (PARTITION BY provider_type, peer_region
                               ORDER BY paid_first_window), 3) AS cohort_percentile
FROM early e
QUALIFY PERCENT_RANK() OVER (PARTITION BY provider_type, peer_region
                             ORDER BY paid_first_window) >= :percentile_cut
    OR days_to_first_claim <= :suspicious_first_claim_days
ORDER BY paid_first_window DESC;

-- days_to_first_claim <= 7 is an independent and arguably stronger signal: an entity
-- billing within a week of enrollment had a patient panel before it was enrolled.
-- QUALIFY is Snowflake/BigQuery/Teradata; elsewhere wrap and filter in an outer WHERE.
```

**4 · Threshold guidance.** :percentile_cut = 0.95 within a same-type, same-region enrollment cohort is more defensible
than any absolute dollar figure because it self-calibrates to your market.
:suspicious_first_claim_days = 7 is the independent signal and often the better one. A high
`beneficiaries_per_active_day` in a brand-new entity is a third - acquiring patients faster
than a new practice plausibly can is the pattern, and it distinguishes a genuine new practice
ramping up from an entity that arrived with a list. Note that a practice acquired as a going
concern will legitimately bill at volume from day one; exclude change-of-ownership enrollments
or they dominate. Tune to your population; a starting point, not a standard.

**5 · Paired control + failure mode.** PE-15 (New-provider billing-velocity watch, first 60-90 days). Secondary: PE-16 (Acquisition-velocity flag), PI-02 (Provider risk-scoring & review-prioritization model).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No velocity watch on newly enrolled providers. PE-15's velocity-alert disposition does not exist. |
| **Designed-not-operating** | PE-15 is continuous; no watch list covers the cohort enrolled in the period. |
| **Operated-but-bypassed** | Velocity alerts triggered no prepayment review, site visit or enhanced screening. |
| **Operated-but-ineffective** | The watch uses a fixed statewide dollar threshold rather than a cohort percentile, missing fast-ramping entities in lower-cost regions and specialties. |
| **Evidence-missing** | No retained watch list or alert dispositions. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 6. Acquisition velocity - one owner accumulating billing numbers

| | |
|---|---|
| **Paired control** | **PE-16** - Acquisition-velocity flag (owner acquiring many numbers fast) |
| **Control type** | Detective |
| **Module** | Module 01 - Provider Enrollment |
| **Owner** | PI data analyst |
| **Frequency** | Monthly |
| **System touchpoint** | Graph + enrollment analytics |
| **Evidence artifact** | Acquisition-velocity alert \| Ownership corroboration via SOS/SAM + CMS disclosure; FinCEN BOI retired 2026-08-14. |
| **Control failure mode** | Mass acquisition of suppliers (Gold Rush) |
| **Regulatory anchor** | DOJ 2025 Operation Gold Rush [case evidence]; Program Integrity Alliance - Value-for-Money in Anti-Fraud Efforts [case evidence] |
| **Secondary controls** | PE-05 - Beneficial-ownership network/graph analysis; PE-06 - 30-day ownership-change enforcement & unreported-change detection |

**1 · Data hypothesis.** The Gold Rush pattern: rather than enrolling new entities, acquire existing ones with clean
histories and established billing numbers. Where acquisition-velocity monitoring is absent, one
owner or control group can accumulate many NPIs quickly and each individual acquisition looks
routine. The signature is only visible in the ownership graph over time.

**2 · What to check.** For each owner identity, count distinct entities acquired within a rolling window, join to the
billing those entities generated post-acquisition, and flag accumulation above the population
norm - especially where acquired entities change service mix or billing address after the
change of ownership.

**3 · Reference SQL.**

```sql
-- TEST 6 - Entities acquired per owner per window, and post-acquisition behaviour change
-- Params: :period_start, :period_end, :window_days (180), :acquisition_threshold (3),
--         :mix_shift_threshold (0.30)
-- Portability: date +/- integer days and date - date (a day count) are PostgreSQL/DuckDB style;
--   elsewhere use DATEADD / DATE_ADD for the offset and DATEDIFF / DATE_DIFF for the difference.
-- Portability: STRING_AGG is PostgreSQL/SQL Server/DuckDB style; elsewhere use LISTAGG (Oracle,
--   Snowflake, Redshift) or GROUP_CONCAT (MySQL).

WITH acquisitions AS (
  SELECT
      oc.owner_identity_hash,
      oc.entity_npi,
      oc.change_effective_date,
      oc.ownership_pct
  FROM ownership_change oc
  WHERE oc.change_effective_date BETWEEN :period_start AND :period_end
    AND oc.ownership_pct >= 5                     -- 5%+ disclosable interest
),
by_owner AS (
  SELECT
      a.owner_identity_hash,
      COUNT(DISTINCT a.entity_npi) AS entities_acquired,
      MIN(a.change_effective_date) AS first_acquisition,
      MAX(a.change_effective_date) AS last_acquisition,
      MAX(a.change_effective_date) - MIN(a.change_effective_date) AS acquisition_span_days,
      STRING_AGG(DISTINCT a.entity_npi, ', ') AS entities
  FROM acquisitions a
  GROUP BY a.owner_identity_hash
  HAVING COUNT(DISTINCT a.entity_npi) >= :acquisition_threshold
)
SELECT
    b.*,
    -- billing generated by the acquired entities AFTER the change of ownership
    (SELECT COALESCE(SUM(cl.paid_amount), 0)
       FROM acquisitions a2
       JOIN claim_lines cl ON cl.billing_npi = a2.entity_npi
                          AND cl.service_date >= a2.change_effective_date
                          AND cl.adjudication_status = 'PAID'
      WHERE a2.owner_identity_hash = b.owner_identity_hash) AS paid_post_acquisition,
    -- entities whose service mix changed materially after acquisition
    (SELECT COUNT(*) FROM acquisitions a3
      WHERE a3.owner_identity_hash = b.owner_identity_hash
        AND EXISTS (
          SELECT 1 FROM entity_service_mix_change m
          WHERE m.entity_npi = a3.entity_npi
            AND m.change_date BETWEEN a3.change_effective_date
                                  AND a3.change_effective_date + 180
            AND m.mix_shift_magnitude > :mix_shift_threshold)) AS entities_with_mix_change
FROM by_owner b
WHERE b.acquisition_span_days <= :window_days
ORDER BY paid_post_acquisition DESC;
```

**4 · Threshold guidance.** :acquisition_threshold = 3 entities within :window_days = 180 is a starting cut and will not
fit every market - consolidation is a legitimate and dominant trend in healthcare, and private-
equity and health-system roll-ups will fill this output. Volume of acquisition is therefore not
the signal on its own. `entities_with_mix_change` is: an acquired entity that abruptly starts
billing different services from a different address is a different proposition from one
absorbed into a group and continuing its practice. Owner-identity resolution is the
prerequisite, and it is hard - corporate layering exists to defeat exactly this join. Tune to
your population; a starting point, not a standard.

**5 · Paired control + failure mode.** PE-16 (Acquisition-velocity flag - owner acquiring many numbers fast). Secondary: PE-05 (Beneficial-ownership network/graph analysis), PE-06 (30-day ownership-change enforcement & unreported-change detection).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No portfolio-level ownership scan exists. PE-16's acquisition-velocity alert does not exist. |
| **Designed-not-operating** | PE-16 is monthly; no scan ran for the period. |
| **Operated-but-bypassed** | Accumulation was identified and no enhanced screening, site visit or review followed. |
| **Operated-but-ineffective** | The scan counts acquisitions per disclosed owner name without identity resolution, so layered ownership through intermediate entities defeats it entirely. |
| **Evidence-missing** | Scans run with no retained alert or disposition record. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 7. Services outside the provider's credentialed scope or taxonomy

| | |
|---|---|
| **Paired control** | **PE-01** - Primary-source verification of all enrollment data before approval |
| **Control type** | Preventive |
| **Module** | Module 01 - Provider Enrollment |
| **Owner** | Enrollment specialist |
| **Frequency** | Per app |
| **System touchpoint** | PECOS / credentialing system |
| **Evidence artifact** | Verification log per data element |
| **Control failure mode** | Fabricated identity/credential enrollment |
| **Regulatory anchor** | 42 CFR 424 Subpart P [binding]; CMS PIM Pub 100-08 Ch.10 [guidance] |
| **Secondary controls** | PE-18 - CAQH ProView primary-source verification & 120-day re-attestation; CB-01 - Certified-coder assignment & coding-to-documentation policy |

**1 · Data hypothesis.** Where primary-source verification of enrollment data is absent or superficial, a provider can
bill services their credential, licence type or taxonomy does not support. This is sometimes
fraud (a fabricated credential), sometimes a rented identity, and often a billing-configuration
error - but in every case the enrollment record and the claim disagree about what this provider
can do.

**2 · What to check.** Join each paid claim to the rendering provider's enrolled taxonomy and licence type, and flag
services outside the scope that taxonomy supports, using a taxonomy-to-service reference you
maintain.

**3 · Reference SQL.**

```sql
-- TEST 7 - Claims for services outside the enrolled taxonomy's scope
-- Requires a taxonomy-to-permitted-service reference. Build it conservatively:
-- over-broad mappings hide real findings, over-narrow ones bury you in false ones.
-- Licence: each service is judged on the licence record in force on ITS service date (the
-- latest record, for the provider's licence type and state, that took effect on or before
-- that date), not on today's record. This needs provider_licence to hold history: one row
-- per issue, renewal or status change, each with the date it took effect.
-- Params: :period_start, :period_end

WITH licence_as_at AS (
  SELECT cl.claim_line_id, l.npi, l.licence_status, l.licence_effective, l.licence_expiry,
         ROW_NUMBER() OVER (PARTITION BY cl.claim_line_id
                            ORDER BY l.licence_effective DESC) AS rn
  FROM claim_lines cl
  JOIN provider pr ON pr.npi = cl.rendering_npi
  JOIN provider_licence l
    ON  l.npi           = cl.rendering_npi
    AND l.licence_type  = pr.licence_type        -- drop these two lines if your licence table
    AND l.licence_state = pr.licence_state       -- does not carry type and state
    AND l.licence_effective <= cl.service_date
  WHERE cl.service_date BETWEEN :period_start AND :period_end
    AND cl.adjudication_status = 'PAID'
)
SELECT
    cl.claim_line_id, cl.rendering_npi, cl.billing_npi,
    cl.hcpcs, cl.service_date, cl.paid_amount,
    pr.primary_taxonomy, pr.licence_type, pr.licence_state,
    CASE WHEN ts.permitted IS NULL THEN 1 ELSE 0 END AS outside_scope,
    lic.licence_status, lic.licence_effective, lic.licence_expiry,
    CASE
      WHEN lic.npi IS NULL                                  THEN 'NO_LICENCE_RECORD'   -- none in force on the service date
      WHEN lic.licence_status <> 'ACTIVE'                   THEN 'LICENCE_NOT_ACTIVE'
      WHEN lic.licence_expiry < cl.service_date             THEN 'LICENCE_EXPIRED'
      WHEN ts.permitted IS NULL                             THEN 'OUTSIDE_TAXONOMY_SCOPE'
    END AS scope_defect
FROM claim_lines cl
JOIN provider pr ON pr.npi = cl.rendering_npi
LEFT JOIN taxonomy_service_scope ts
       ON  ts.taxonomy_code = pr.primary_taxonomy
       AND ts.hcpcs         = cl.hcpcs
-- the expiry is tested in the CASE and WHERE, not in the join, so a licence that had
-- already expired on the service date is still found and labelled
LEFT JOIN licence_as_at lic
       ON  lic.claim_line_id = cl.claim_line_id
       AND lic.rn = 1
WHERE cl.service_date BETWEEN :period_start AND :period_end
  AND cl.adjudication_status = 'PAID'
  AND (ts.permitted IS NULL
       OR lic.npi IS NULL
       OR lic.licence_status <> 'ACTIVE'
       OR lic.licence_expiry < cl.service_date)
ORDER BY scope_defect, cl.paid_amount DESC;

-- Roll up by rendering_npi AND by hcpcs. A single code failing for many providers is a
-- taxonomy-mapping defect in YOUR reference table. Fix that before reviewing anyone.
```

**4 · Threshold guidance.** The licence checks are categorical, and each service is judged on the licence record in force
on its service date. They are only as reliable as your licence history: if the licence table
keeps just the current record, with no dated rows for renewals and status changes, past
services are judged on today's record and both false flags and misses follow, so confirm that
history exists before relying on LICENCE_NOT_ACTIVE or LICENCE_EXPIRED. The taxonomy-scope
check is only as good as your mapping, which is why the roll-up instruction matters. Scope of
practice varies by state, and taxonomy codes are self-selected at enrollment and frequently
inaccurate - a provider with a stale taxonomy is common and is an enrollment-data finding
(PE-02), not a fraud one. Start with a small, high-confidence set of clearly-out-of-scope pairs
(surgical procedures under non-physician taxonomies, for instance) rather than attempting
comprehensive mapping. Tune to your population; a starting point, not a standard.

**5 · Paired control + failure mode.** PE-01 (Primary-source verification of all enrollment data before approval). Secondary: PE-18 (CAQH ProView primary-source verification & 120-day re-attestation), CB-01 (Certified-coder assignment & coding-to-documentation policy).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No primary-source verification at enrollment. PE-01's per-element verification log does not exist. |
| **Designed-not-operating** | PE-01 is per-application; a population of approved enrollments has no verification log. |
| **Operated-but-bypassed** | Verification returned a discrepancy and the enrollment was approved anyway. |
| **Operated-but-ineffective** | Verification confirms the credential exists but never re-checks it after approval, so a subsequent revocation or expiry is invisible until revalidation. |
| **Evidence-missing** | Verification performed by a delegated credentialing entity with no evidence surfaced to the payer. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 8. Enrollment-of-record staleness - claims contradicting the enrollment file

| | |
|---|---|
| **Paired control** | **PE-02** - PECOS/enrollment-of-record data reconciliation cycle |
| **Control type** | Detective |
| **Module** | Module 01 - Provider Enrollment |
| **Owner** | Enrollment data steward |
| **Frequency** | Quarterly |
| **System touchpoint** | PECOS extract + QA tool |
| **Evidence artifact** | Reconciliation exception report |
| **Control failure mode** | Stale/inaccurate record enabling fraud |
| **Regulatory anchor** | OIG OEI-03-13-00050 (Enhanced Enrollment Screening of Medicare Providers; 27 Apr 2016) [case evidence] |
| **Secondary controls** | PE-06 - 30-day ownership-change enforcement & unreported-change detection; MD-16 - T-MSIS monthly submission + data-quality validation |

**1 · Data hypothesis.** Every test in this family joins claims to the enrollment record, so the enrollment record's
accuracy bounds all of them. Where the reconciliation cycle is absent, the enrollment file
drifts from reality - addresses that no longer exist, practice locations closed, banking
details changed without notice, entities merged - and PE-02's stated failure mode,
'stale/inaccurate record enabling fraud', is realised quietly.

**2 · What to check.** Profile the enrollment file against the claim stream and external sources: NPIs with claims but
no active enrollment, enrollments with no claims for extended periods, addresses that fail
validation, and records whose external-registry status contradicts the enrollment status.

**3 · Reference SQL.**

```sql
-- TEST 8 - Enrollment-of-record reconciliation defect profile
-- Run before tests 2, 4, 5, 6 and 7 - all of them join to this record.
-- Params: :period_start, :period_end, :silent_enrollment_days (540)
-- Portability: date +/- integer days and date - date (a day count) are PostgreSQL/DuckDB style;
--   elsewhere use DATEADD / DATE_ADD for the offset and DATEDIFF / DATE_DIFF for the difference.

WITH defects AS (
  SELECT cl.billing_npi AS npi, 'CLAIMS_NO_ACTIVE_ENROLLMENT' AS defect,
         SUM(cl.paid_amount) AS paid_amount
  FROM claim_lines cl
  LEFT JOIN provider_enrollment e ON e.npi = cl.billing_npi AND e.enrollment_status = 'ACTIVE'
  WHERE cl.service_date BETWEEN :period_start AND :period_end
    AND cl.adjudication_status = 'PAID'
    AND e.npi IS NULL
  GROUP BY cl.billing_npi

  UNION ALL

  SELECT e.npi, 'ACTIVE_ENROLLMENT_NO_CLAIMS', 0
  FROM provider_enrollment e
  WHERE e.enrollment_status = 'ACTIVE'
    AND NOT EXISTS (SELECT 1 FROM claim_lines cl
                    WHERE cl.billing_npi = e.npi
                      AND cl.service_date > CURRENT_DATE - :silent_enrollment_days)

  UNION ALL

  SELECT e.npi, 'ADDRESS_FAILS_VALIDATION', 0
  FROM provider_enrollment e
  JOIN address_validation av ON av.address_id = e.practice_address_id
  WHERE e.enrollment_status = 'ACTIVE' AND av.validation_result <> 'VALID'

  UNION ALL

  SELECT e.npi, 'NPPES_STATUS_CONTRADICTS_ENROLLMENT', 0
  FROM provider_enrollment e
  JOIN nppes_snapshot n ON n.npi = e.npi
  WHERE e.enrollment_status = 'ACTIVE'
    AND (n.npi_deactivation_date IS NOT NULL
         OR n.entity_type <> e.expected_entity_type)

  UNION ALL

  SELECT e.npi, 'BANKING_CHANGED_NO_DISCLOSURE', 0
  FROM provider_enrollment e
  JOIN eft_change_log b ON b.npi = e.npi
  WHERE b.change_date BETWEEN :period_start AND :period_end
    AND NOT EXISTS (SELECT 1 FROM ownership_change oc
                    WHERE oc.entity_npi = e.npi
                      AND oc.change_effective_date BETWEEN b.change_date - 60 AND b.change_date + 60)
)
SELECT defect, COUNT(DISTINCT npi) AS affected_npis, SUM(paid_amount) AS paid_behind_defect
FROM defects
GROUP BY defect
ORDER BY paid_behind_defect DESC, affected_npis DESC;
```

**4 · Threshold guidance.** Each defect class is categorical; materiality is the judgement. Two deserve particular
attention. CLAIMS_NO_ACTIVE_ENROLLMENT should be near zero in a functioning system - a non-
trivial count means either the enrollment join key is wrong or claims are paying for unenrolled
entities, and both are serious in different ways. BANKING_CHANGED_NO_DISCLOSURE is the highest-
value single check here: an EFT change without a corresponding ownership or contact-change
filing is the payment-diversion pattern that OIG raised an open recommendation on in 2025, and
it is cheap to run. Tune to your population; a starting point, not a standard.

**5 · Paired control + failure mode.** PE-02 (PECOS/enrollment-of-record data reconciliation cycle). Secondary: PE-06 (30-day ownership-change enforcement & unreported-change detection), MD-16 (T-MSIS submission + data-quality validation).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No reconciliation cycle. PE-02's reconciliation exception report does not exist. |
| **Designed-not-operating** | PE-02 is quarterly; one or more quarterly runs did not occur. |
| **Operated-but-bypassed** | Exceptions were reported and never corrected at source, so the same defects recur each quarter. |
| **Operated-but-ineffective** | Reconciliation checks internal consistency only and never compares against NPPES, address validation or banking-change data - so external contradictions are invisible. |
| **Evidence-missing** | No retained exception report or evidence of correction. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 9. Ownership network - entities sharing owners, addresses, or banking details

| | |
|---|---|
| **Paired control** | **PE-05** - Beneficial-ownership network/graph analysis |
| **Control type** | Detective |
| **Module** | Module 01 - Provider Enrollment |
| **Owner** | PI data analyst |
| **Frequency** | Monthly |
| **System touchpoint** | Graph analytics platform |
| **Evidence artifact** | Network-link alert disposition |
| **Control failure mode** | Shell-company rings (Gold Rush) |
| **Regulatory anchor** | DOJ 2025 Operation Gold Rush [case evidence]; Program Integrity Alliance - Transparency by Design (company registry / single identifier) [case evidence] |
| **Secondary controls** | PE-04 - Owner reconciliation vs. record + OIG/SAM/adverse databases; MD-03 - Ownership/control & business-transaction disclosure capture + … |

**1 · Data hypothesis.** Shell-company rings share attributes even when they hide ownership: a registered agent, a suite
number, a bank account, a phone, an officer. Where beneficial-ownership graph analysis is
absent, the entities look independent and each passes screening on its own. PE-05 cites Gold
Rush, where the network structure was the case.

**2 · What to check.** Build an entity graph from disclosed ownership plus shared address, banking, contact and
officer attributes, find connected components, and report clusters with more entities than
disclosed ownership alone would predict - weighted by the billing the cluster generates.

**3 · Reference SQL.**

```sql
-- TEST 9 - Entity clusters linked by shared attributes beyond disclosed ownership
-- Full graph work belongs in a graph platform (PE-05 names one). This SQL builds the
-- edge list and a one-hop cluster; take it to the graph tool for components and centrality.
-- Edges are built in both directions, so EVERY member of a cluster is returned with its own
-- row. cluster_size counts the entity itself plus the entities directly linked to it.
-- Params: :period_start, :period_end, :min_cluster_size (3)
-- Portability: STRING_AGG is PostgreSQL/SQL Server/DuckDB style; elsewhere use LISTAGG (Oracle,
--   Snowflake, Redshift) or GROUP_CONCAT (MySQL).

WITH edges AS (
  SELECT a.entity_npi AS entity_a, b.entity_npi AS entity_b, 'SHARED_ADDRESS' AS link_type
  FROM entity_attribute a JOIN entity_attribute b
    ON a.address_hash = b.address_hash AND a.entity_npi <> b.entity_npi
   AND a.address_hash NOT IN (SELECT address_hash FROM shared_address_exclusion)
  UNION
  SELECT a.entity_npi, b.entity_npi, 'SHARED_BANK_ACCOUNT'
  FROM entity_attribute a JOIN entity_attribute b
    ON a.bank_account_hash = b.bank_account_hash AND a.entity_npi <> b.entity_npi
  UNION
  SELECT a.entity_npi, b.entity_npi, 'SHARED_PHONE'
  FROM entity_attribute a JOIN entity_attribute b
    ON a.phone_hash = b.phone_hash AND a.entity_npi <> b.entity_npi
   AND a.phone_hash NOT IN (SELECT phone_hash FROM shared_phone_exclusion)
  UNION
  SELECT o1.entity_npi, o2.entity_npi, 'SHARED_OWNER'
  FROM ownership_disclosure o1 JOIN ownership_disclosure o2
    ON o1.owner_identity_hash = o2.owner_identity_hash AND o1.entity_npi <> o2.entity_npi
  UNION
  SELECT a.entity_npi, b.entity_npi, 'SHARED_REGISTERED_AGENT'
  FROM entity_attribute a JOIN entity_attribute b
    ON a.registered_agent_hash = b.registered_agent_hash AND a.entity_npi <> b.entity_npi
   AND a.registered_agent_hash NOT IN (SELECT agent_hash FROM commercial_agent_exclusion)
),
clusters AS (
  SELECT entity_a AS entity_npi,
         COUNT(DISTINCT entity_b)     AS linked_entities,
         COUNT(DISTINCT entity_b) + 1 AS cluster_size,
         COUNT(DISTINCT link_type)    AS distinct_link_types,
         STRING_AGG(DISTINCT link_type, ', ') AS link_types
  FROM edges GROUP BY entity_a
)
SELECT
    c.entity_npi, c.cluster_size, c.linked_entities, c.distinct_link_types, c.link_types,
    (SELECT COALESCE(SUM(cl.paid_amount), 0) FROM claim_lines cl
      WHERE cl.billing_npi = c.entity_npi
        AND cl.service_date BETWEEN :period_start AND :period_end
        AND cl.adjudication_status = 'PAID') AS entity_paid,
    (SELECT COUNT(*) FROM ownership_disclosure od WHERE od.entity_npi = c.entity_npi)
      AS disclosed_owners
FROM clusters c
WHERE c.cluster_size >= :min_cluster_size
ORDER BY c.distinct_link_types DESC, entity_paid DESC;
```

**4 · Threshold guidance.** :min_cluster_size = 3 entities (the entity itself plus two linked directly to it), and every
member of a cluster is listed. `distinct_link_types` is the discriminating column - two
entities sharing an address is a medical office building, two entities sharing an address *and*
a bank account *and* a phone is something else. Rank on link-type diversity, not edge count.
The exclusion lists are mandatory rather than optional: commercial registered agents serve
thousands of entities, medical office buildings and billing companies legitimately share
addresses and phones, and without those exclusions the largest cluster in your data will be a
registered-agent artifact. Tune to your population; a starting point, not a standard.

**5 · Paired control + failure mode.** PE-05 (Beneficial-ownership network/graph analysis). Secondary: PE-04 (Owner reconciliation vs. record + OIG/SAM/adverse databases), MD-03 (Ownership/control & business-transaction disclosure capture + verification).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No graph or network analysis exists. PE-05's network-link alert disposition does not exist. |
| **Designed-not-operating** | PE-05 is monthly; no portfolio scan ran in the period. |
| **Operated-but-bypassed** | Network links were surfaced and no enhanced screening or investigation followed. |
| **Operated-but-ineffective** | The analysis uses disclosed ownership only and no shared-attribute edges, so networks that hide ownership - which is the point of a shell ring - are invisible to it. |
| **Evidence-missing** | Analysis performed with no retained alert or disposition record. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 10. Undisclosed ownership or control change within the reporting window

| | |
|---|---|
| **Paired control** | **PE-06** - 30-day ownership-change enforcement & unreported-change detection |
| **Control type** | Detective |
| **Module** | Module 01 - Provider Enrollment |
| **Owner** | Enrollment ops |
| **Frequency** | Continuous |
| **System touchpoint** | PECOS + external data feed |
| **Evidence artifact** | Change-report tracking log |
| **Control failure mode** | Undisclosed control changes |
| **Regulatory anchor** | 42 CFR 424.516 [binding]; OIG OEI-07-23-00180 (25 Feb 2025) - EFT payment-diversion fraud; rec 25-E-07-017.03 open, .01 and .02 closed [case evidence] |
| **Secondary controls** | MD-03 - Ownership/control & business-transaction disclosure capture + …; PE-04 - Owner reconciliation vs. record + OIG/SAM/adverse databases |

**1 · Data hypothesis.** 42 CFR 424.516 requires reporting of ownership and control changes within 30 days. Where
enforcement and unreported-change detection are absent, entities change hands silently - and a
change of control without disclosure is how an excluded or adverse party takes over a clean
billing number. PE-06 also cites the 2025 OIG EFT payment-diversion recommendation, which is
the same failure seen from the banking side.

**2 · What to check.** Compare the enrollment ownership record against external sources - Secretary of State filings,
corporate registries, banking changes - and flag entities where an external change has no
corresponding disclosure within the reporting window.

**3 · Reference SQL.**

```sql
-- TEST 10 - External ownership/control change with no corresponding disclosure
-- 42 CFR 424.516 (30-day reporting).
-- Params: :period_start, :period_end, :report_window_days (30)
-- Portability: date +/- integer days and date - date (a day count) are PostgreSQL/DuckDB style;
--   elsewhere use DATEADD / DATE_ADD for the offset and DATEDIFF / DATE_DIFF for the difference.

SELECT
    ext.entity_npi,
    ext.source,                       -- 'SOS_FILING' | 'EFT_CHANGE' | 'REGISTRY' | 'SAM'
    ext.change_type,                  -- 'OFFICER' | 'OWNER' | 'AGENT' | 'BANKING' | 'NAME'
    ext.change_date,
    ext.old_value_hash, ext.new_value_hash,
    d.disclosure_id, d.disclosure_date,
    CASE
      WHEN d.disclosure_id IS NULL THEN 'NO_DISCLOSURE'
      WHEN d.disclosure_date > ext.change_date + :report_window_days THEN 'LATE_DISCLOSURE'
      ELSE 'TIMELY'
    END AS disclosure_status,
    COALESCE(d.disclosure_date, CURRENT_DATE) - ext.change_date AS days_to_disclosure,
    (SELECT COALESCE(SUM(cl.paid_amount), 0) FROM claim_lines cl
      WHERE cl.billing_npi = ext.entity_npi
        AND cl.service_date > ext.change_date
        AND cl.adjudication_status = 'PAID') AS paid_since_change
FROM external_entity_change ext
-- earliest disclosure on or after the change, whenever it was filed; the CASE then
-- separates timely from late (a join limited to the window could never see a late one)
LEFT JOIN (SELECT e.entity_npi, e.change_date, MIN(o.disclosure_date) AS first_disclosure_date
           FROM external_entity_change e
           JOIN ownership_disclosure o
             ON o.entity_npi = e.entity_npi AND o.disclosure_date >= e.change_date
           GROUP BY e.entity_npi, e.change_date) fd
       ON  fd.entity_npi = ext.entity_npi AND fd.change_date = ext.change_date
LEFT JOIN ownership_disclosure d
       ON  d.entity_npi = fd.entity_npi
       AND d.disclosure_date = fd.first_disclosure_date
WHERE ext.change_date BETWEEN :period_start AND :period_end
  AND (d.disclosure_id IS NULL
       OR d.disclosure_date > ext.change_date + :report_window_days)
ORDER BY paid_since_change DESC;

-- Cross-check every NO_DISCLOSURE hit against the exclusion lists for the NEW owner or
-- officer. An undisclosed change of control TO an excluded party is a different and much
-- more serious finding than a late filing, and it is the reason the rule exists.
```

**4 · Threshold guidance.** The 30-day window is regulatory, not tunable. What is tunable is which external sources you
trust and how you resolve their entity identifiers to NPIs, and that resolution is the hard
part - Secretary of State records use legal entity names that map imperfectly to enrolled
provider names. Expect a meaningful false-positive rate from name-matching alone and treat
unresolved matches as candidates. The `paid_since_change` column sizes the exposure; the
exclusion cross-check noted in the query is what separates an administrative lapse from the
case the rule exists to prevent. Tune to your population; a starting point, not a standard.

**5 · Paired control + failure mode.** PE-06 (30-day ownership-change enforcement & unreported-change detection). Secondary: MD-03 (Ownership/control & business-transaction disclosure capture + verification), PE-04 (Owner reconciliation vs. record + OIG/SAM/adverse databases).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No unreported-change detection. PE-06's change-report tracking log does not exist. |
| **Designed-not-operating** | PE-06 specifies continuous monitoring with monthly external-feed reconciliation; no reconciliation ran. |
| **Operated-but-bypassed** | Undisclosed changes were identified and no enrollment action, re-screening or payment hold followed. |
| **Operated-but-ineffective** | The control tracks disclosures received but never reconciles against an external feed, so it can only detect late filings, never absent ones. |
| **Evidence-missing** | No retained tracking log evidencing which changes were reconciled. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 11. Multi-source link analysis - cross-provider scheme convergence

| | |
|---|---|
| **Paired control** | **PI-03** - Multi-source link analysis (claims+enrollment+encounter+external) |
| **Control type** | Detective |
| **Module** | Module 02 - Program Integrity |
| **Owner** | PI data analyst |
| **Frequency** | Monthly |
| **System touchpoint** | Integrated data platform |
| **Evidence artifact** | Network-link alert disposition |
| **Control failure mode** | Cross-provider schemes missed |
| **Regulatory anchor** | CMS PIM Pub 100-08 Ch.2 (Data Analysis) [guidance] |
| **Secondary controls** | PE-05 - Beneficial-ownership network/graph analysis; PI-22 - Entity-network cash-loop detection - follow-the-money analytics … |

**1 · Data hypothesis.** Individual providers in a coordinated scheme each look ordinary. The scheme is visible only
where claims, enrollment, encounter and external data converge: shared beneficiary panels,
sequential referral chains, common staff moving between entities, synchronised billing
patterns. Where multi-source link analysis is absent, cross-provider schemes are missed by
construction - PI-03's stated failure mode.

**2 · What to check.** Build provider-to-provider edges from beneficiary overlap, referral flow and shared rendering
personnel, and surface tightly-connected components whose internal linkage far exceeds what
geography and specialty would predict.

**3 · Reference SQL.**

```sql
-- TEST 11 - Provider convergence: shared panels, referral chains, shared personnel
-- Edge list for a graph platform.
-- Params: :period_start, :period_end, :min_shared_beneficiaries (25), :jaccard_threshold (0.30)

WITH panel_overlap AS (
  SELECT
      a.billing_npi AS provider_a,
      b.billing_npi AS provider_b,
      COUNT(DISTINCT a.beneficiary_id) AS shared_beneficiaries
  FROM (SELECT DISTINCT billing_npi, beneficiary_id FROM claim_lines
        WHERE service_date BETWEEN :period_start AND :period_end
          AND adjudication_status = 'PAID') a
  JOIN (SELECT DISTINCT billing_npi, beneficiary_id FROM claim_lines
        WHERE service_date BETWEEN :period_start AND :period_end
          AND adjudication_status = 'PAID') b
    ON a.beneficiary_id = b.beneficiary_id AND a.billing_npi < b.billing_npi
  GROUP BY a.billing_npi, b.billing_npi
  HAVING COUNT(DISTINCT a.beneficiary_id) >= :min_shared_beneficiaries
),
panel_sizes AS (
  SELECT billing_npi, COUNT(DISTINCT beneficiary_id) AS panel_size
  FROM claim_lines
  WHERE service_date BETWEEN :period_start AND :period_end AND adjudication_status = 'PAID'
  GROUP BY billing_npi
),
shared_staff AS (
  SELECT a.billing_npi AS provider_a, b.billing_npi AS provider_b,
         COUNT(DISTINCT a.rendering_npi) AS shared_renderers
  FROM (SELECT DISTINCT billing_npi, rendering_npi FROM claim_lines
        WHERE service_date BETWEEN :period_start AND :period_end) a
  JOIN (SELECT DISTINCT billing_npi, rendering_npi FROM claim_lines
        WHERE service_date BETWEEN :period_start AND :period_end) b
    ON a.rendering_npi = b.rendering_npi AND a.billing_npi < b.billing_npi
  GROUP BY a.billing_npi, b.billing_npi
)
SELECT
    po.provider_a, po.provider_b,
    po.shared_beneficiaries,
    pa.panel_size AS panel_a, pb.panel_size AS panel_b,
    -- Jaccard: overlap relative to combined panel size, not raw count
    ROUND(po.shared_beneficiaries * 1.0 /
          NULLIF(pa.panel_size + pb.panel_size - po.shared_beneficiaries, 0), 3) AS jaccard,
    COALESCE(ss.shared_renderers, 0) AS shared_renderers
FROM panel_overlap po
JOIN panel_sizes pa ON pa.billing_npi = po.provider_a
JOIN panel_sizes pb ON pb.billing_npi = po.provider_b
LEFT JOIN shared_staff ss ON ss.provider_a = po.provider_a AND ss.provider_b = po.provider_b
WHERE po.shared_beneficiaries * 1.0 /
      NULLIF(pa.panel_size + pb.panel_size - po.shared_beneficiaries, 0) > :jaccard_threshold
ORDER BY jaccard DESC, po.shared_beneficiaries DESC;
```

**4 · Threshold guidance.** Use the Jaccard coefficient rather than the raw overlap count - two large practices in one city
will share hundreds of patients and mean nothing by it, while two small entities sharing 80% of
a panel mean a great deal. :jaccard_threshold = 0.30 is a starting cut. Referral relationships,
hospital-affiliated groups, ACOs and integrated delivery systems all produce high overlap
legitimately and should be excluded via a known-affiliation list before the first run.
`shared_renderers` is the second dimension worth ranking on: shared patients plus shared staff
plus no disclosed affiliation is the interesting conjunction. Tune to your population; a
starting point, not a standard.

**5 · Paired control + failure mode.** PI-03 (Multi-source link analysis - claims + enrollment + encounter + external). Secondary: PE-05 (Beneficial-ownership network/graph analysis), PI-22 (Entity-network cash-loop detection).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No link analysis capability. PI-03's network-link alert disposition does not exist. |
| **Designed-not-operating** | PI-03 is monthly; no scheme-detection scan ran. |
| **Operated-but-bypassed** | Networks were surfaced and no case was opened or referral made. |
| **Operated-but-ineffective** | The analysis runs on claims only, so enrollment, ownership and external attributes never enter the graph and the links that define a scheme are unavailable to it. |
| **Evidence-missing** | Analysis run with no retained output or disposition. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 12. Credentialing re-attestation lapse in payer networks

| | |
|---|---|
| **Paired control** | **PE-18** - CAQH ProView primary-source verification & 120-day re-attestation |
| **Control type** | Preventive |
| **Module** | Module 01 - Provider Enrollment |
| **Owner** | Credentialing team |
| **Frequency** | Per app + 120 days |
| **System touchpoint** | CAQH ProView |
| **Evidence artifact** | Attestation/PSV record |
| **Control failure mode** | Stale credentialing data (payer networks) |
| **Regulatory anchor** | CAQH ProView / NCQA credentialing [guidance] |
| **Secondary controls** | PE-01 - Primary-source verification of all enrollment data before approval; MC-11 - FDR inventory, pre-contract screening & periodic audit |

**1 · Data hypothesis.** Payer network credentialing depends on providers re-attesting to the currency of their data on
a defined cycle. Where re-attestation is not enforced, network directories and credentialing
files carry stale data - PE-18's stated failure mode - and the payer's assurance about who is
in its network degrades quietly while claims continue to pay.

**2 · What to check.** For each network provider, compute days since last attestation and last primary-source
verification, flag those beyond the cycle, and attach the claims paid during the lapsed
interval to size the exposure.

**3 · Reference SQL.**

```sql
-- TEST 12 - Attestation and PSV currency across the network
-- Params: :period_start, :period_end, :attestation_cycle_days (120)
-- Portability: date +/- integer days and date - date (a day count) are PostgreSQL/DuckDB style;
--   elsewhere use DATEADD / DATE_ADD for the offset and DATEDIFF / DATE_DIFF for the difference.

SELECT
    c.npi, c.provider_name_hash, c.network_id, c.specialty,
    c.last_attestation_date,
    CURRENT_DATE - c.last_attestation_date AS days_since_attestation,
    c.last_psv_date,
    CURRENT_DATE - c.last_psv_date         AS days_since_psv,
    c.licence_expiry, c.dea_expiry, c.malpractice_expiry,
    CASE
      WHEN c.licence_expiry   < CURRENT_DATE                          THEN 'LICENCE_EXPIRED'
      WHEN c.malpractice_expiry < CURRENT_DATE                        THEN 'COVERAGE_EXPIRED'
      WHEN c.last_attestation_date IS NULL                            THEN 'NEVER_ATTESTED'
      WHEN CURRENT_DATE - c.last_attestation_date > :attestation_cycle_days
                                                                      THEN 'ATTESTATION_LAPSED'
    END AS credential_defect,
    (SELECT COALESCE(SUM(cl.paid_amount), 0) FROM claim_lines cl
      WHERE cl.rendering_npi = c.npi
        AND cl.service_date > COALESCE(c.last_attestation_date, DATE '1900-01-01')
                              + :attestation_cycle_days
        AND cl.service_date BETWEEN :period_start AND :period_end
        AND cl.adjudication_status = 'PAID') AS paid_during_lapse
FROM credentialing_record c
WHERE c.network_status = 'ACTIVE'
  AND (c.last_attestation_date IS NULL
       OR CURRENT_DATE - c.last_attestation_date > :attestation_cycle_days
       OR c.licence_expiry     < CURRENT_DATE
       OR c.malpractice_expiry < CURRENT_DATE)
ORDER BY paid_during_lapse DESC;
```

**4 · Threshold guidance.** :attestation_cycle_days = 120 matches the CAQH ProView cadence PE-18 names; confirm against
your own delegated-credentialing agreements, which may differ. Note what this test does and
does not show: an attestation lapse is an assurance failure, not evidence that anything is
wrong with the provider, and `paid_during_lapse` sizes the volume of claims paid without
current assurance rather than an overpayment. Expired licence and expired malpractice coverage
are materially more serious than a late attestation and should be reported separately rather
than as one defect count. Tune to your population; a starting point, not a standard.

**5 · Paired control + failure mode.** PE-18 (CAQH ProView primary-source verification & 120-day re-attestation). Secondary: PE-01 (Primary-source verification of enrollment data), MC-11 (FDR inventory, pre-contract screening & periodic audit).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No re-attestation cycle operates. PE-18's attestation/PSV record does not exist. |
| **Designed-not-operating** | The cycle is defined but a population of active network providers is past due. |
| **Operated-but-bypassed** | Lapsed providers remained in-network and payable with no restriction applied. |
| **Operated-but-ineffective** | Attestation is collected but not primary-source verified, so the control confirms the provider said something rather than that it is true. |
| **Evidence-missing** | Credentialing is delegated and no attestation evidence is retained by the payer. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 13. Synthetic-identity indicators at enrollment

| | |
|---|---|
| **Paired control** | **PE-20** - Synthetic-identity & deepfake identity-proofing at enrollment / high-risk change - liveness & deepfake check on submitted identity documents plus cross-registry consistency (NPPES/PECOS/LEIE/SAM) to catch AI-fabricated provider or beneficiary identities |
| **Control type** | Preventive |
| **Module** | Module 01 - Provider Enrollment |
| **Owner** | Enrollment specialist / credentialing |
| **Frequency** | Per app / high-risk change |
| **System touchpoint** | PECOS / credentialing intake + ID-verification service |
| **Evidence artifact** | Identity-proofing log (liveness result + registry cross-match) per applicant |
| **Control failure mode** | AI-fabricated (synthetic or deepfaked) provider/beneficiary identity passing screening |
| **Regulatory anchor** | 42 CFR 455.410/.436 (Medicaid provider screening) [binding]; 42 CFR 424.500-.570 (Medicare enrollment & identity) [binding]; NIST AI RMF Generative AI Profile (synthetic-content risk) [guidance] |
| **Secondary controls** | PE-01 - Primary-source verification of all enrollment data before approval; PE-13 - Fingerprint-based background checks (5%+ owners, High-risk) |

**1 · Data hypothesis.** PE-20 exists because AI-fabricated and synthetic provider identities can pass document-based
screening. Where identity-proofing is absent, a synthetic identity enrolls successfully and
every downstream control then operates faithfully on a person who does not exist. The data
signature is thin history: an identity with a credential but no corroborating footprint
anywhere else.

**2 · What to check.** For each recently enrolled individual provider, count corroborating records across independent
sources - NPPES history, state licence board, DEA registry, medical school and residency
verification, prior claims history under any payer, Open Payments - and flag identities
corroborated by few or none.

**3 · Reference SQL.**

```sql
-- TEST 13 - Corroboration thinness for recently enrolled individual providers
-- A synthetic identity is characterised by ABSENCE across independent sources.
-- This is a screening-quality test; it does not identify fraud on its own.
-- Params: :period_start, :period_end, :min_corroborating_sources (3)
-- Portability: date +/- integer days and date - date (a day count) are PostgreSQL/DuckDB style;
--   elsewhere use DATEADD / DATE_ADD for the offset and DATEDIFF / DATE_DIFF for the difference.

WITH recent AS (
  SELECT e.npi, e.enrollment_effective_date, e.provider_type, e.risk_level
  FROM provider_enrollment e
  WHERE e.enrollment_effective_date BETWEEN :period_start AND :period_end
    AND e.entity_type = 'INDIVIDUAL'
)
SELECT
    r.npi, r.enrollment_effective_date, r.provider_type, r.risk_level,
    CASE WHEN n.npi IS NOT NULL AND n.enumeration_date < r.enrollment_effective_date - 365
         THEN 1 ELSE 0 END AS nppes_history,
    CASE WHEN lic.npi IS NOT NULL THEN 1 ELSE 0 END AS state_licence_verified,
    CASE WHEN dea.npi IS NOT NULL THEN 1 ELSE 0 END AS dea_registered,
    CASE WHEN edu.npi IS NOT NULL THEN 1 ELSE 0 END AS education_verified,
    CASE WHEN hist.npi IS NOT NULL THEN 1 ELSE 0 END AS prior_claim_history,
    CASE WHEN op.physician_npi IS NOT NULL THEN 1 ELSE 0 END AS open_payments_presence,
    CASE WHEN idp.npi IS NOT NULL THEN 1 ELSE 0 END AS identity_proofing_performed,
    (CASE WHEN n.npi IS NOT NULL AND n.enumeration_date < r.enrollment_effective_date - 365 THEN 1 ELSE 0 END
     + CASE WHEN lic.npi  IS NOT NULL THEN 1 ELSE 0 END
     + CASE WHEN dea.npi  IS NOT NULL THEN 1 ELSE 0 END
     + CASE WHEN edu.npi  IS NOT NULL THEN 1 ELSE 0 END
     + CASE WHEN hist.npi IS NOT NULL THEN 1 ELSE 0 END
     + CASE WHEN op.physician_npi IS NOT NULL THEN 1 ELSE 0 END) AS corroborating_sources,
    (SELECT COALESCE(SUM(cl.paid_amount), 0) FROM claim_lines cl
      WHERE cl.rendering_npi = r.npi AND cl.adjudication_status = 'PAID') AS paid_to_date
FROM recent r
LEFT JOIN nppes_snapshot n       ON n.npi = r.npi
LEFT JOIN provider_licence lic   ON lic.npi = r.npi AND lic.licence_status = 'ACTIVE'
LEFT JOIN dea_registration dea   ON dea.npi = r.npi
LEFT JOIN education_verification edu ON edu.npi = r.npi
LEFT JOIN (SELECT DISTINCT rendering_npi AS npi FROM claim_lines
            WHERE service_date < :period_start) hist ON hist.npi = r.npi
LEFT JOIN open_payments_summary op ON op.physician_npi = r.npi
LEFT JOIN identity_proofing_log idp ON idp.npi = r.npi
WHERE (CASE WHEN n.npi IS NOT NULL AND n.enumeration_date < r.enrollment_effective_date - 365 THEN 1 ELSE 0 END
     + CASE WHEN lic.npi  IS NOT NULL THEN 1 ELSE 0 END
     + CASE WHEN dea.npi  IS NOT NULL THEN 1 ELSE 0 END
     + CASE WHEN edu.npi  IS NOT NULL THEN 1 ELSE 0 END
     + CASE WHEN hist.npi IS NOT NULL THEN 1 ELSE 0 END
     + CASE WHEN op.physician_npi IS NOT NULL THEN 1 ELSE 0 END) < :min_corroborating_sources
ORDER BY corroborating_sources ASC, paid_to_date DESC;
```

**4 · Threshold guidance.** :min_corroborating_sources = 3 is a starting cut and must be read with real care, because thin
corroboration is the normal condition of a genuinely new provider. A newly-licensed clinician
has no prior claim history, no Open Payments presence, possibly no DEA registration, and a
recent NPPES enumeration - they will score low and be entirely legitimate. This test is
therefore not a fraud detector; it is a measure of whether identity-proofing (the
`identity_proofing_performed` column) was applied where corroboration was thin. Read those two
columns together: thin corroboration *and* no identity proofing is the PE-20 finding. Thin
corroboration with proofing performed is the control working. Tune to your population; a
starting point, not a standard.

**5 · Paired control + failure mode.** PE-20 (Synthetic-identity & deepfake identity-proofing at enrollment / high-risk change). Secondary: PE-01 (Primary-source verification of enrollment data), PE-13 (Fingerprint-based background checks).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No identity proofing at enrollment. PE-20's identity-proofing log does not exist. |
| **Designed-not-operating** | PE-20 is per-application; a population of approved individual enrollments has no proofing record. |
| **Operated-but-bypassed** | Proofing returned an inconclusive or adverse result and the enrollment proceeded. |
| **Operated-but-ineffective** | Proofing verifies documents rather than liveness and registry cross-match, which is precisely the check a synthetic or deepfaked identity is built to pass. |
| **Evidence-missing** | Proofing performed by a vendor with no per-applicant result retained. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 14. Downstream entity rendering services but absent from the FDR inventory

| | |
|---|---|
| **Paired control** | **MC-11** - FDR inventory, pre-contract screening & periodic audit |
| **Control type** | Preventive |
| **Module** | Module 06 - Managed Care (Part C/D) |
| **Owner** | Vendor/FDR oversight lead |
| **Frequency** | Annual |
| **System touchpoint** | FDR register + audit tool |
| **Evidence artifact** | FDR audit report |
| **Control failure mode** | Downstream entity fraud unmonitored (423.504 F) |
| **Regulatory anchor** | 42 CFR Part 423 (Medicare Part D) [binding]; 42 CFR 423.504(b)(4)(vi) [binding] |
| **Secondary controls** | PE-18 - CAQH ProView primary-source verification & 120-day re-attestation; MD-10 - MCO contract PI safeguards: FWA reporting, overpayment recovery, PI … |

**1 · Data hypothesis.** Plans are accountable for first-tier, downstream and related entities. Where the FDR inventory
is incomplete, entities render or bill services with no screening, no training and no audit
coverage - MC-11's stated failure mode, 'downstream entity fraud unmonitored'. The inventory is
an assertion; the claim stream is the fact against which to test it.

**2 · What to check.** Extract every distinct entity appearing as billing, rendering or referring party on the plan's
claims and encounters, and reconcile against the FDR register and the contracted-provider file.
Report entities present in the data but absent from both.

**3 · Reference SQL.**

```sql
-- TEST 14 - Entities in the claim stream absent from the FDR / contracted-provider registers
-- Params: :period_start, :period_end
-- Portability: date +/- integer days and date - date (a day count) are PostgreSQL/DuckDB style;
--   elsewhere use DATEADD / DATE_ADD for the offset and DATEDIFF / DATE_DIFF for the difference.

WITH entities_in_data AS (
  SELECT DISTINCT plan_npi, billing_npi AS entity_npi, 'BILLING' AS role
  FROM claim_lines
  WHERE service_date BETWEEN :period_start AND :period_end AND adjudication_status = 'PAID'
  UNION
  SELECT DISTINCT plan_npi, rendering_npi, 'RENDERING'
  FROM claim_lines
  WHERE service_date BETWEEN :period_start AND :period_end AND adjudication_status = 'PAID'
    AND rendering_npi IS NOT NULL
  UNION
  SELECT DISTINCT plan_npi, vendor_npi, 'DELEGATED_VENDOR'
  FROM delegated_service_log
  WHERE service_date BETWEEN :period_start AND :period_end
)
SELECT
    e.plan_npi, e.entity_npi, e.role,
    fdr.fdr_id, fdr.screening_date, fdr.last_audit_date,
    cp.contract_id, cp.contract_effective,
    CASE
      WHEN fdr.fdr_id IS NULL AND cp.contract_id IS NULL THEN 'NOT_IN_ANY_REGISTER'
      WHEN fdr.fdr_id IS NULL                            THEN 'CONTRACTED_NOT_IN_FDR_REGISTER'
      WHEN fdr.screening_date IS NULL                    THEN 'IN_REGISTER_NEVER_SCREENED'
      WHEN fdr.last_audit_date IS NULL
        OR fdr.last_audit_date < CURRENT_DATE - 365      THEN 'AUDIT_OVERDUE'
      WHEN cp.contract_id IS NULL
       AND e.role <> 'DELEGATED_VENDOR'                  THEN 'IN_FDR_REGISTER_NOT_CONTRACTED'
    END AS register_defect,
    (SELECT COALESCE(SUM(cl.paid_amount), 0) FROM claim_lines cl
      WHERE cl.billing_npi = e.entity_npi
        AND cl.service_date BETWEEN :period_start AND :period_end
        AND cl.adjudication_status = 'PAID') AS entity_paid
FROM entities_in_data e
LEFT JOIN fdr_register fdr      ON fdr.entity_npi = e.entity_npi AND fdr.plan_npi = e.plan_npi
LEFT JOIN contracted_provider cp ON cp.provider_npi = e.entity_npi AND cp.plan_npi = e.plan_npi
WHERE fdr.fdr_id IS NULL
   OR (cp.contract_id IS NULL AND e.role <> 'DELEGATED_VENDOR')   -- a screened, audited FDR vendor needs no provider contract
   OR fdr.screening_date IS NULL
   OR fdr.last_audit_date IS NULL
   OR fdr.last_audit_date < CURRENT_DATE - 365
ORDER BY register_defect, entity_paid DESC;
```

**4 · Threshold guidance.** Categorical against the registers, but the register definitions do real work here. Not every
entity in a claim stream is an FDR - out-of-network emergency providers, one-off referrals and
non-contracted ancillary providers legitimately appear without a contract or an FDR entry, and
NOT_IN_ANY_REGISTER will be dominated by them unless out-of-network claims are excluded first.
The genuinely interesting classes are IN_REGISTER_NEVER_SCREENED and AUDIT_OVERDUE, which are
about the plan's own execution against entities it has already acknowledged. Start there. Tune
to your population; a starting point, not a standard.

**5 · Paired control + failure mode.** MC-11 (FDR inventory, pre-contract screening & periodic audit). Secondary: PE-18 (CAQH ProView PSV & re-attestation), MD-10 (MCO contract PI safeguards).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No FDR register exists. MC-11's FDR audit report does not exist. |
| **Designed-not-operating** | The register exists but no annual audit cycle ran in the period. |
| **Operated-but-bypassed** | Entities were identified as unscreened and continued to render services with no remediation. |
| **Operated-but-ineffective** | The register is populated from contracts only and never reconciled against the claim stream, so entities operating without a contract are invisible to it. |
| **Evidence-missing** | Audits performed by the FDR itself with no evidence retained by the plan. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 15. Entity-network cash loop - claim proceeds recycling among related parties

| | |
|---|---|
| **Paired control** | **PI-22** - Entity-network cash-loop detection - follow-the-money analytics flagging entity clusters that recycle claim proceeds into cash used to induce recipients/providers (layered inter-entity transfers) |
| **Control type** | Detective |
| **Module** | Module 02 - Program Integrity |
| **Owner** | SIU / financial analytics |
| **Frequency** | Quarterly |
| **System touchpoint** | Claims + payment + ownership/entity data |
| **Evidence artifact** | Entity-network transfer-pattern alert + dispositions |
| **Control failure mode** | Laundering entities converting claim proceeds into bribe cash (self-funding loop in a 2026 adult day-care scheme, alleged) |
| **Regulatory anchor** | 42 USC 1320a-7b(b) (Anti-Kickback Statute) [binding]; 18 USC 1956 (laundering of monetary instruments) [binding] |
| **Secondary controls** | PE-05 - Beneficial-ownership network/graph analysis; PI-23 - MSO/investment-disguised kickback detection - cross-reference … |

**1 · Data hypothesis.** Where follow-the-money analytics are absent, entity clusters can recycle claim proceeds through
inter-entity transfers inconsistent with legitimate operations - converting payments into cash
for kickbacks or extraction. PI-22 cites an adult day-care self-funding loop from the
enforcement record. The signature is circularity: money leaving an entity and returning to the
network it came from.

**2 · What to check.** Build a payment graph across related entities from claim payments and inter-entity transfers,
and detect cycles - money paid to entity A flowing to B and back toward A's owners or A itself
 -  weighted by the share of receipts that recirculate.

**3 · Reference SQL.**

```sql
-- TEST 15 - Circular value flow within an entity cluster
-- Requires inter-entity payment data (vendor payments, management fees, rent, payroll to
-- related parties). Many payers do not hold this; where it exists it is usually obtained
-- in an investigation rather than standing.
-- Params: :period_start, :period_end, :recirculation_threshold (0.20)

WITH receipts AS (
  SELECT cl.billing_npi AS entity_npi, SUM(cl.paid_amount) AS claim_receipts
  FROM claim_lines cl
  WHERE cl.service_date BETWEEN :period_start AND :period_end
    AND cl.adjudication_status = 'PAID'
  GROUP BY cl.billing_npi
),
transfers AS (
  SELECT t.payer_entity_npi, t.payee_entity_npi, t.transfer_category,
         SUM(t.amount) AS amount
  FROM inter_entity_transfer t
  WHERE t.transfer_date BETWEEN :period_start AND :period_end
  GROUP BY t.payer_entity_npi, t.payee_entity_npi, t.transfer_category
),
two_hop_cycles AS (
  -- A -> B -> A, and A -> B -> C where C shares an owner with A
  SELECT
      t1.payer_entity_npi AS origin,
      t1.payee_entity_npi AS hop1,
      t2.payee_entity_npi AS hop2,
      t1.amount AS leg1_amount, t2.amount AS leg2_amount,
      t1.transfer_category AS leg1_category, t2.transfer_category AS leg2_category,
      CASE WHEN t2.payee_entity_npi = t1.payer_entity_npi THEN 'DIRECT_LOOP'
           ELSE 'OWNER_LINKED_LOOP' END AS loop_type
  FROM transfers t1
  JOIN transfers t2 ON t2.payer_entity_npi = t1.payee_entity_npi
  WHERE t2.payee_entity_npi = t1.payer_entity_npi
     OR EXISTS (SELECT 1 FROM ownership_disclosure o1
                JOIN ownership_disclosure o2
                  ON o1.owner_identity_hash = o2.owner_identity_hash
                WHERE o1.entity_npi = t1.payer_entity_npi
                  AND o2.entity_npi = t2.payee_entity_npi)
)
SELECT
    c.origin, c.hop1, c.hop2, c.loop_type,
    c.leg1_category, c.leg2_category,
    c.leg1_amount, c.leg2_amount,
    r.claim_receipts,
    ROUND(LEAST(c.leg1_amount, c.leg2_amount) / NULLIF(r.claim_receipts, 0), 3)
      AS recirculated_share
FROM two_hop_cycles c
JOIN receipts r ON r.entity_npi = c.origin
WHERE LEAST(c.leg1_amount, c.leg2_amount) / NULLIF(r.claim_receipts, 0) > :recirculation_threshold
ORDER BY recirculated_share DESC, r.claim_receipts DESC;
```

**4 · Threshold guidance.** :recirculation_threshold = 0.20 of claim receipts recirculating through a two-hop loop is a
starting cut. Be clear about what this test can and cannot be: inter-entity transfers are
overwhelmingly legitimate - management fees, rent to a related property company, shared
services, payroll through a PEO - and circularity in a commonly-owned group is normal corporate
structure, not evidence of anything. The `transfer_category` columns matter more than the
amounts: loops running through categories with no plausible service content are the ones worth
asking about. Most payers will not hold this data at all, in which case that absence is the
PI-22 finding and the query is a specification for what investigators would need to request.
Tune to your population; a starting point, not a standard.

**5 · Paired control + failure mode.** PI-22 (Entity-network cash-loop detection). Secondary: PE-05 (Beneficial-ownership network/graph analysis), PI-23 (MSO/investment-disguised kickback detection).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No follow-the-money capability. PI-22's entity-network transfer-pattern alert does not exist. |
| **Designed-not-operating** | PI-22 is quarterly; no analysis ran in the period. |
| **Operated-but-bypassed** | Loops were identified and no investigation, referral or legal assessment followed. |
| **Operated-but-ineffective** | The analysis runs on claim payments only, with no inter-entity transfer data, so it can see money going out and never where it goes next - which is the entire question. |
| **Evidence-missing** | Analysis performed inside an investigation with no standing alert record or disposition. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 16. AI-fabricated supporting documentation - corroboration mismatch

| | |
|---|---|
| **Paired control** | **PI-24** - AI-fabricated-evidence corroboration analytics - treat submitted documentation as a claim to be independently confirmed: verify the service/encounter actually occurred from source-obtained data the submitter does not control (EVV, dispensing, scheduling, cross-payer) rather than from the document itself |
| **Control type** | Detective |
| **Module** | Module 02 - Program Integrity |
| **Owner** | SIU / program-integrity analytics |
| **Frequency** | Continuous / per review cycle |
| **System touchpoint** | Claims + external corroboration feeds |
| **Evidence artifact** | Corroboration-mismatch exception report |
| **Control failure mode** | Payment on AI-generated fabricated documentation that passes visual audit |
| **Regulatory anchor** | 42 CFR 455 Subpart A (Medicaid fraud detection) [binding]; NIST AI RMF (AI 100-1) + Generative AI Profile (content provenance / corroboration) [guidance] |
| **Secondary controls** | CD-20 - AI-generated / altered document & image provenance screening - check …; CD-07 - Copy-forward / templated-note control: EHR authorship controls limit … |

**1 · Data hypothesis.** PI-24 states the principle directly: treat submitted documentation as a claim to be
independently corroborated rather than as evidence. Where that control is absent, AI-generated
records that pass visual and structural review support payment. Because fabricated
documentation is internally consistent by construction, the only reliable signal is external:
the document asserts facts that independent data contradicts or cannot confirm.

**2 · What to check.** For documentation submitted in support of high-risk or audit-flagged claims, test each
externally checkable assertion - the ordering provider existed and was active, the referenced
prior encounter exists, the referenced test result has a corresponding lab claim, the signing
clinician was licensed on the signature date - and report documents with assertions that fail
corroboration.

**3 · Reference SQL.**

```sql
-- TEST 16 - Documentation assertions that independent data cannot corroborate
-- Pair with CD-20 (C2PA / Content Credentials provenance screening), which tests the
-- ARTIFACT; this tests the CLAIMS the artifact makes. Both are needed.
-- Params: :period_start, :period_end

SELECT
    d.document_id, d.claim_id, d.document_type, d.received_date,
    d.asserted_author_npi, d.asserted_signature_date,
    d.asserted_prior_encounter_date, d.asserted_test_result_date,
    -- corroboration checks, each independently verifiable
    CASE WHEN p.npi IS NULL THEN 0 ELSE 1 END AS author_npi_exists,
    CASE WHEN lic.npi IS NOT NULL
          AND d.asserted_signature_date BETWEEN lic.licence_effective
                                            AND COALESCE(lic.licence_expiry, DATE '9999-12-31')
         THEN 1 ELSE 0 END AS author_licensed_on_signature_date,
    CASE WHEN p.date_of_death IS NOT NULL
          AND d.asserted_signature_date > p.date_of_death
         THEN 1 ELSE 0 END AS signed_after_author_death,
    CASE WHEN EXISTS (SELECT 1 FROM claim_lines e
                      WHERE e.beneficiary_id = d.beneficiary_id
                        AND e.rendering_npi  = d.asserted_author_npi
                        AND e.service_date   = d.asserted_prior_encounter_date
                        AND e.adjudication_status = 'PAID')
         THEN 1 ELSE 0 END AS prior_encounter_corroborated,
    CASE WHEN EXISTS (SELECT 1 FROM claim_lines l
                      WHERE l.beneficiary_id = d.beneficiary_id
                        AND l.service_date   = d.asserted_test_result_date
                        AND l.hcpcs IN (SELECT hcpcs FROM lab_service_code_list)
                        AND l.adjudication_status = 'PAID')
         THEN 1 ELSE 0 END AS test_result_corroborated,
    prov.provenance_result, prov.c2pa_present
FROM submitted_document d
LEFT JOIN provider p        ON p.npi = d.asserted_author_npi
LEFT JOIN provider_licence lic ON lic.npi = d.asserted_author_npi
LEFT JOIN document_provenance prov ON prov.document_id = d.document_id
WHERE d.received_date BETWEEN :period_start AND :period_end
  AND (p.npi IS NULL
       OR (p.date_of_death IS NOT NULL AND d.asserted_signature_date > p.date_of_death)
       OR (d.asserted_prior_encounter_date IS NOT NULL      -- only test what the document asserts
           AND NOT EXISTS (SELECT 1 FROM claim_lines e
                      WHERE e.beneficiary_id = d.beneficiary_id
                        AND e.rendering_npi  = d.asserted_author_npi
                        AND e.service_date   = d.asserted_prior_encounter_date
                        AND e.adjudication_status = 'PAID'))
       OR (d.asserted_test_result_date IS NOT NULL
           AND NOT EXISTS (SELECT 1 FROM claim_lines l
                      WHERE l.beneficiary_id = d.beneficiary_id
                        AND l.service_date   = d.asserted_test_result_date
                        AND l.hcpcs IN (SELECT hcpcs FROM lab_service_code_list)
                        AND l.adjudication_status = 'PAID')))
ORDER BY d.received_date DESC;
```

**4 · Threshold guidance.** Each corroboration check is binary, but they are not equal. `signed_after_author_death` and a
non-existent author NPI are near-conclusive. An uncorroborated prior encounter or test result
is much weaker: care delivered under another payer, self-paid services and simple date
transcription errors all produce it, and a document describing real care will frequently fail
these checks. Count the failures rather than acting on any single one, and read the result
alongside CD-20's provenance screening - an absent C2PA manifest proves nothing on its own
(most legitimate documents lack one today), but absent provenance combined with multiple failed
corroborations is a coherent picture. Tune to your population; a starting point, not a
standard.

**5 · Paired control + failure mode.** PI-24 (AI-fabricated-evidence corroboration analytics). Secondary: CD-20 (AI-generated / altered document & image provenance screening), CD-07 (Copy-forward / templated-note control).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | Submitted documentation is accepted at face value. PI-24's corroboration-mismatch exception report does not exist. |
| **Designed-not-operating** | PI-24 is continuous / per review cycle; no corroboration was performed on documentation received in the period. |
| **Operated-but-bypassed** | Corroboration failed and the claim was paid on the strength of the document anyway. |
| **Operated-but-ineffective** | Review inspects the document for internal consistency and legibility - exactly the properties generated documentation has in abundance - and never tests its assertions against independent data. |
| **Evidence-missing** | Reviews occurred with no retained record of which assertions were corroborated. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## Gap notes - controls this family needs that the library does not yet have

*Each note names a control objective these tests point to that the control library does not yet state. They are candidates for a future library release; the library itself is unchanged by this pack.*

### G1. No control addresses supervision, incident-to, or split/shared billing conditions.

Incident-to billing is a rule about who was physically present and what relationship existed
between the billing physician and the person who furnished the service. The control library has
controls about documentation authorship (CD-06), documentation sufficiency (CD-03) and billing-
identity consistency (PI-20), but nothing that states the supervision condition as a control
objective. Test 3 pairs to CD-06 by proximity and its 'Absent' row says so explicitly. This
matters because incident-to is high-volume, the payment differential is substantial, and the
service is usually genuinely delivered - which makes it a rate-integrity failure that no
services-not-rendered control will ever catch.

**Recommendation.** A supervision-conditions control for Module 03 or Module 05: verification that incident-to and
split/shared billing meet the applicable supervision, plan-of-care and setting conditions, with
the evidence artifact naming what demonstrates physician presence. Note the rules differ by
setting and have been revised repeatedly, so the control should reference current requirements
rather than restate them.

### G2. PI-20 covers footprint inconsistency but not credential sharing as such.

PI-20's design - claims from a location or entity inconsistent with enrollment - catches the
rented credential when the renter operates somewhere else. It does not address the case where
one clinician's identity is used by several people at the same location, which produces no
geographic inconsistency at all and shows up only as implausible volume (test 1) or as an
incident-to pattern (test 3). No control states the objective that the NPI on a claim
identifies the individual who personally performed the service.

**Recommendation.** Consider whether this is best handled by widening PI-20's stated objective or by a companion
control. Either way the analytic pairing for tests 1 and 3 would improve materially - both
currently pair to controls (CB-08, CD-06) written for other purposes.

### G3. Several tests in this family require reference data most payers do not hold.

Test 15 (cash loops) needs inter-entity transfer data, which payers generally obtain during an
investigation rather than as standing data. Test 10 needs external corporate-registry feeds.
Test 6 needs resolved owner identity across layered structures. Test 9 needs entity attributes
(banking, registered agent) beyond what enrollment captures. These are not defects in the
queries - the controls themselves (PE-05, PE-06, PE-16, PI-22) name these data sources - but a
payer will find a large part of this family unrunnable on day one.

**Recommendation.** Where a query cannot be run because the data does not exist, that absence is itself the control
finding, and each affected test's threshold guidance now says so. Worth stating once in the
preface as a general principle rather than only test-by-test: an unrunnable query in this pack
is a data-availability finding about the control, not a gap in the pack.

### G4. Module 01 (Provider Enrollment) controls are testable from claims, and few were.

In the first four families of this pack only three Module 01 (Provider Enrollment) controls
carry a query: PE-08, PE-09 and PE-15. Module 01 has 20 controls and is one of the strongest
modules in the library, but its controls are largely preventive and operate at enrollment,
which makes them feel like a poor fit for claims analytics. This family shows they are not:
enrollment controls are testable from claims precisely because a failed enrollment control
leaves a claim that contradicts the enrollment record.

**Recommendation.** No control change is needed. This family adds queries for PE-01, PE-02, PE-05, PE-06, PE-14,
PE-16, PE-18 and PE-20, so that 11 of the 20 Module 01 controls have a paired query. The
remaining nine are mostly site-visit and fingerprinting controls whose evidence is genuinely
not in the claim stream.

---

*Reference patterns. Public-source, personal-capacity, advisory and organizational only. Queries must be adapted to your own schema and tested on your own warehouse before use. Outputs are review candidates for human adjudication, never payment or denial dispositions. Confirm every regulatory anchor against its primary source.*