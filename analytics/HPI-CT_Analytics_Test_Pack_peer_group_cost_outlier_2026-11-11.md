# Analytics Test Pack - Peer-group cost-per-beneficiary outlier - and the program around it

*MEVIA-style vulnerability detection: the z-score, the peer group that makes it valid, and the loop that makes it worth running*

*Generated 2026-11-11 from the HPI-CT control library (`data/controls.csv`, 173 controls). 16 detection tests, paired to 16 primary controls. Reference patterns, not programs.*

## Before you run anything

**These queries produce review candidates, never payment or denial decisions.** Every hit is a question for a human.

**The toolkit never receives your data.** These are patterns you run inside your own environment. No claim data, no PHI, no identifiers leave your systems.

**Every query is a pattern, not a program.** Field names are generic; yours will differ. Expect to rewrite the FROM and JOIN clauses and to remap service categories to your own taxonomy.

**Every threshold is a starting point, not a standard.** Where a threshold is arithmetic it is labelled as such. Everywhere else, measure your own distribution first.

Read `PREFACE_how_to_run_these.md` before the first run.

## Scope of this family

Provider-level, pharmacy-level, facility-level and plan-level spend outliers measured against a
risk-adjusted peer group - the general-purpose surveillance layer that sits above the six
scheme-specific families already in this pack. Scope covers both halves of an outlier program:
the detection statistics themselves (tests 1-8) and the triage, feedback and closure loop that
decides whether a flag ever becomes a finding (tests 9-16). The reference point is MEVIA - the
Medicaid Emerging Vulnerability Intelligence and Actions hub - which is CMS productising
exactly this loop: detect a vulnerability centrally, hand it to a state, and track whether the
state acted.

This family is deliberately different from families 1-6 and should be read that way. The
earlier families each detect a named scheme with a specific data signature; a hit is close to
self-explanatory. A peer-group outlier explains nothing on its own - it is a ranking, and a
ranking is only as good as the peer group, the risk adjustment and the estimator behind it. So
half of this family tests the statistics and half tests the program: whether the flags get
worked, whether the interventions changed anything, whether the same provider is being audited
three times by three contractors, and whether anyone is measuring how often the model is wrong.
The second half is the part most outlier programs never build, and it is where MEVIA's actual
contribution lies - not the z-score, which is arithmetic anyone can run, but the closed loop
from intelligence to owned action. Tests 9-16 therefore run against the payer's own case-
management and audit tracking systems rather than against claims; that split is called out on
every test.

## Tests at a glance

| # | Test | Primary control | Also touches |
|---|---|---|---|
| 1 | Risk-adjusted cost-per-beneficiary peer z-score | **PI-01** | PI-02, MD-14 |
| 2 | Peer-group construction integrity - taxonomy misassignment and thin peer cells | **PE-02** | PI-01, MD-16 |
| 3 | Outlier decomposition - is it price, volume, or panel? | **PI-02** | PI-01, CB-19 |
| 4 | Estimator fragility - parametric z-score versus robust median/MAD | **AG-01** | PI-01, PI-02 |
| 5 | Pharmacy cost-per-beneficiary outlier and high-cost drug concentration | **MC-10** | MC-13, PI-01 |
| 6 | Beneficiary-side utilisation outlier and cross-provider concentration | **MC-14** | PI-21, MD-14 |
| 7 | Institutional cost-per-day outlier and cost-report-to-claims divergence | **MD-23** | PI-01, MD-11 |
| 8 | Capitation and directed-payment outlier - dollars per member the acuity does not explain | **MD-11** | MD-17, MC-24 |
| 9 | Documentation-request aging and non-response - flags that never became evidence | **PI-13** | PI-05, CD-14 |
| 10 | Flag-to-disposition yield - precision, cycle time and return by test | **PI-19** | PI-02, AG-01 |
| 11 | Cross-programme duplication - the same provider-period audited more than once | **PI-15** | PI-13, PI-19 |
| 12 | Did the intervention work? Pre/post re-measurement of the flagged behaviour | **PI-11** | PI-10, MD-13 |
| 13 | Education without effect - repeat outliers across consecutive detection cycles | **PI-12** | PI-11, CB-19 |
| 14 | Identified but not recovered - overpayment aging against the look-back limit | **PI-09** | PI-07, MC-19 |
| 15 | Payment velocity during an open investigation - exposure accruing while a case runs | **PI-16** | MD-18, PI-14 |
| 16 | Detection coverage map - which spend has no test looking at it | **MD-19** | PI-01, PI-18 |

> **Control-ID validation:** every control referenced above resolves to a real ID in the current control library. See the gap notes at the end for controls this family *needs* but the library does not yet have.

---

## 1. Risk-adjusted cost-per-beneficiary peer z-score

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
| **Secondary controls** | PI-02 - Provider risk-scoring & review-prioritization model; MD-14 - Utilization control & SURS medical-review program |

**1 · Data hypothesis.** If routine aberrancy analytics are absent, run only on raw totals, or run without risk
adjustment, a provider whose per-beneficiary cost sits far above its specialty and regional
peers can bill at that level indefinitely without anyone asking why. Almost every scheme in the
case base eventually shows up here, because almost every scheme ends in money: whether the
mechanism is upcoding, phantom services, unnecessary services or unit-splitting, the terminal
signature is the same - this provider costs more per patient than providers doing the same work
on comparable patients. That generality is the test's strength and its weakness. It catches
nearly everything and explains nearly nothing, which is why it is a triage instrument and never
a finding.

**2 · What to check.** For each rendering or billing provider, compute total paid dollars divided by the count of
distinct beneficiaries served in the period. Normalise for case mix using whatever risk score
your population already carries (HCC, CDPS, CRG, ACG). Group providers into peer cells by
specialty/taxonomy and region, compute the cell mean and standard deviation, and express each
provider as a z-score. Return the upper tail with enough context columns - panel size, claim
count, dollars - that a reviewer can see immediately whether the outlier is a real volume story
or a small-denominator artifact.

**3 · Reference SQL.**

```sql
-- TEST 1 - Risk-adjusted cost-per-beneficiary peer z-score (the MEVIA-style core test)
-- Pattern only. Run on your own warehouse. Output = review candidates, not dispositions.
-- Params: :period_start, :period_end, :min_panel_size (30), :min_peer_cell (20), :z_cut (2.5)
-- Portability: STDDEV here is the sample standard deviation; the standard SQL name is STDDEV_SAMP.

WITH provider_panel AS (
  SELECT
      cl.billing_npi,
      cl.rendering_npi,
      pr.specialty_taxonomy,                       -- from YOUR enrollment-of-record, not the claim
      pr.region_code,                              -- county, MSA or rating region - pick one and keep it
      COUNT(DISTINCT cl.beneficiary_id)            AS panel_size,
      COUNT(*)                                     AS line_count,
      SUM(cl.paid_amount)                          AS paid_total,
      -- risk weight of the panel: mean of whatever risk score you already compute.
      -- If you have none, run the test UNADJUSTED and label it so - do not invent one here.
      AVG(bx.risk_score)                           AS mean_risk_score
  FROM claim_lines cl
  LEFT JOIN beneficiary_risk bx
         ON bx.beneficiary_id = cl.beneficiary_id
        AND bx.risk_year      = EXTRACT(YEAR FROM cl.service_date)
  JOIN provider_reference pr
         ON pr.npi = cl.rendering_npi
  WHERE cl.service_date BETWEEN :period_start AND :period_end
    AND cl.adjudication_status = 'PAID'
  GROUP BY cl.billing_npi, cl.rendering_npi, pr.specialty_taxonomy, pr.region_code
),
adjusted AS (
  SELECT
      *,
      paid_total / NULLIF(panel_size, 0)                                AS cost_per_bene,
      -- risk-adjusted: divide by the panel's mean risk score so a sicker panel is not an outlier
      (paid_total / NULLIF(panel_size, 0)) / NULLIF(mean_risk_score, 0) AS adj_cost_per_bene
  FROM provider_panel
  WHERE panel_size >= :min_panel_size              -- small denominators produce fake outliers
),
peer AS (
  SELECT
      specialty_taxonomy,
      region_code,
      COUNT(*)                        AS peer_n,
      AVG(adj_cost_per_bene)          AS peer_mean,
      STDDEV(adj_cost_per_bene)       AS peer_sd
  FROM adjusted
  GROUP BY specialty_taxonomy, region_code
  HAVING COUNT(*) >= :min_peer_cell                -- a peer group of four is not a peer group
)
SELECT
    a.rendering_npi,
    a.billing_npi,
    a.specialty_taxonomy,
    a.region_code,
    a.panel_size,
    a.line_count,
    a.paid_total,
    ROUND(a.cost_per_bene, 2)                                     AS cost_per_bene,
    ROUND(a.adj_cost_per_bene, 2)                                 AS adj_cost_per_bene,
    ROUND(p.peer_mean, 2)                                         AS peer_mean,
    p.peer_n,
    ROUND((a.adj_cost_per_bene - p.peer_mean)
          / NULLIF(p.peer_sd, 0), 2)                              AS z_score,
    ROUND(a.adj_cost_per_bene / NULLIF(p.peer_mean, 0), 2)        AS ratio_to_peer_mean
FROM adjusted a
JOIN peer p
  ON p.specialty_taxonomy = a.specialty_taxonomy
 AND p.region_code        = a.region_code
WHERE (a.adj_cost_per_bene - p.peer_mean) / NULLIF(p.peer_sd, 0) > :z_cut
ORDER BY z_score DESC, a.paid_total DESC;

-- Read the ratio column before the z-score. A z of 4.0 in a tight peer group can be a
-- 15% cost difference; a z of 2.6 in a loose one can be triple the peer mean. The z tells
-- you how unusual, the ratio tells you how much money. Reviewers respond to the ratio.
```

**4 · Threshold guidance.** :z_cut = 2.5 is the conventional starting point, but it is the least defensible number in this
pack - a z-score assumes an approximately normal peer distribution, and healthcare cost
distributions are right-skewed with heavy tails, so a fixed z will over-flag in some
specialties and under-flag in others. Measure your own distribution per peer cell before
adopting any cut, and see test 4 for the robust cross-check. :min_panel_size = 30 and
:min_peer_cell = 20 matter more than the cut itself: most spurious outliers in practice are
denominator artifacts, not statistical extremes. Tune to your population; a starting point, not
a standard.

**5 · Paired control + failure mode.** **PI-01 - routine claims-aberrancy analytics (peer-outlier / statistical).** This test is PI-01 executing. A hit is not by itself a control finding: the control finding is what the *absence* of hits, or the absence of anyone looking at them, tells you. Secondary PI-02 (risk-scoring and review prioritisation) is where the hit is supposed to go next; MD-14 (SURS) is the Medicaid statutory equivalent.

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No peer-outlier analytics run at all - the first time this query is executed it produces years of accumulated tail, which is itself the finding. Expect a large initial hit list and resist the urge to tune it down; triage by dollars first. |
| **Designed-not-operating** | An outlier report exists in the PI plan but has not been produced for the period under review, or is produced and never distributed. Check the report's last run date before concluding anything about the providers. |
| **Operated-but-bypassed** | The analytics run but named provider groups are excluded from the population - high-volume systems, contracted networks, or providers with an existing relationship. Compare the query's provider count against the paid-claims provider count for the same period; the gap is the bypass. |
| **Operated-but-ineffective** | The test runs on raw totals with no risk adjustment or no peer grouping, so the top of the list is simply the largest practices. If your top 20 outliers are also your top 20 by volume, the test is measuring size, not aberrancy. |
| **Evidence-missing** | Outliers were identified but no record exists of what was done with them - no disposition, no reviewer, no date. See test 10; an outlier program with no disposition trail cannot demonstrate it operated. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 2. Peer-group construction integrity - taxonomy misassignment and thin peer cells

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
| **Secondary controls** | PI-01 - Routine claims-aberrancy analytics (peer-outlier / statistical); MD-16 - T-MSIS monthly submission + data-quality validation |

**1 · Data hypothesis.** Every peer-comparison test inherits the quality of the enrollment record that defines the peer
group. If a provider's specialty taxonomy in the enrollment file is wrong, stale, or self-
reported and never verified, that provider is compared against the wrong peers - and the
comparison fails in whichever direction the error runs. A high-cost specialist coded as a
generalist becomes a permanent outlier and consumes review capacity forever; a generalist coded
into a high-cost specialty becomes permanently invisible, which is the more dangerous direction
and the one no one checks. Thin peer cells do the same damage by a different route: in a cell
of six, one member's own values dominate the mean it is being compared against.

**2 · What to check.** Three passes. First, a coherence pass: for each provider, compare the specialty taxonomy on the
enrollment record against the specialty implied by its own billing - the procedure mix it
actually submits - and flag providers whose billed mix sits far from the mix of the other
providers in their recorded specialty. Second, a structural pass: count peer cells by size and
report what share of providers and of dollars sit in cells too small to support a peer
statistic at all. Third, a list of providers with no reference row or no recorded specialty,
who cannot be placed in any peer group. Report all three before publishing any outlier list.

**3 · Reference SQL.**

```sql
-- TEST 2 - Peer-group construction integrity: is the peer group real?
-- Three independent passes. Run ALL of them before you publish any outlier ranking.
-- Params: :period_start, :period_end, :min_peer_cell (20), :mix_divergence_cut (0.60), :min_mix_providers (5)
-- Portability: GROUP BY 1 in pass B is an ordinal; repeat the CASE expression where ordinals are not accepted.

-- PASS A - taxonomy coherence: does the recorded specialty match the billed procedure mix?
WITH provider_mix AS (
  SELECT
      cl.rendering_npi,
      pr.specialty_taxonomy                  AS recorded_specialty,
      cl.hcpcs_cpt,
      SUM(cl.paid_amount)                    AS code_paid
  FROM claim_lines cl
  JOIN provider_reference pr ON pr.npi = cl.rendering_npi
  WHERE cl.service_date BETWEEN :period_start AND :period_end
    AND cl.adjudication_status = 'PAID'
    -- a missing specialty has no peer mix to be measured against, so it is not scored
    -- here at all; those providers are listed by pass C
    AND pr.specialty_taxonomy IS NOT NULL
  GROUP BY cl.rendering_npi, pr.specialty_taxonomy, cl.hcpcs_cpt
),
provider_total AS (
  SELECT rendering_npi, recorded_specialty, SUM(code_paid) AS prov_paid
  FROM provider_mix
  GROUP BY rendering_npi, recorded_specialty
),
specialty_code AS (
  SELECT recorded_specialty, hcpcs_cpt, SUM(code_paid) AS spec_code_paid
  FROM provider_mix
  GROUP BY recorded_specialty, hcpcs_cpt
),
specialty_total AS (
  SELECT recorded_specialty, COUNT(*) AS spec_providers, SUM(prov_paid) AS spec_paid
  FROM provider_total
  GROUP BY recorded_specialty
),
compared AS (
  -- leave-one-out: the reference mix is the specialty WITHOUT this provider, so a provider
  -- cannot pull the reference towards its own mix and hide behind it
  SELECT
      pm.rendering_npi,
      pm.recorded_specialty,
      st.spec_providers,
      pm.code_paid / NULLIF(pt.prov_paid, 0)                                      AS prov_share,
      (sc.spec_code_paid - pm.code_paid) / NULLIF(st.spec_paid - pt.prov_paid, 0) AS peer_share
  FROM provider_mix pm
  JOIN provider_total pt
    ON pt.rendering_npi = pm.rendering_npi AND pt.recorded_specialty = pm.recorded_specialty
  JOIN specialty_code sc
    ON sc.recorded_specialty = pm.recorded_specialty AND sc.hcpcs_cpt = pm.hcpcs_cpt
  JOIN specialty_total st
    ON st.recorded_specialty = pm.recorded_specialty
),
scored AS (
  SELECT
      rendering_npi,
      recorded_specialty,
      spec_providers,
      -- total variation distance between this provider's mix and the mix of the other
      -- providers in its specialty: 0 = identical, 1 = no overlap at all. The first SUM
      -- covers the codes the provider billed; "1 - SUM(peer_share)" adds the peer mass on
      -- codes the provider never billed, so the result is a true 0..1 metric. Not scored
      -- in a specialty with fewer than :min_mix_providers providers.
      CASE WHEN spec_providers >= :min_mix_providers
           THEN 0.5 * (SUM(ABS(prov_share - peer_share)) + 1 - SUM(peer_share))
      END                                    AS mix_divergence
  FROM compared
  GROUP BY rendering_npi, recorded_specialty, spec_providers
)
SELECT
    rendering_npi,
    recorded_specialty,
    spec_providers,
    ROUND(mix_divergence, 3)                 AS mix_divergence,
    CASE WHEN spec_providers < :min_mix_providers THEN 'TOO_THIN_TO_COMPARE'
         ELSE 'DIVERGENT' END                AS mix_status
FROM scored
WHERE spec_providers < :min_mix_providers
   OR mix_divergence > :mix_divergence_cut
ORDER BY mix_status, mix_divergence DESC;

-- TOO_THIN_TO_COMPARE is a statement about the specialty, not the provider: with two or
-- three providers the reference mix is one or two practices, and a distance from it says
-- which of them is the odd one only by assumption. Those providers are listed, not scored.

-- PASS B - structural: how much of the population cannot be peer-compared at all?
WITH cell AS (
  SELECT pr.specialty_taxonomy, pr.region_code,
         COUNT(DISTINCT cl.rendering_npi) AS peer_n,
         SUM(cl.paid_amount)              AS cell_paid
  FROM claim_lines cl
  JOIN provider_reference pr ON pr.npi = cl.rendering_npi
  WHERE cl.service_date BETWEEN :period_start AND :period_end
    AND cl.adjudication_status = 'PAID'
  GROUP BY pr.specialty_taxonomy, pr.region_code
)
SELECT
    CASE WHEN peer_n < :min_peer_cell THEN 'UNCOMPARABLE' ELSE 'COMPARABLE' END AS cell_status,
    COUNT(*)                                    AS cells,
    SUM(peer_n)                                 AS providers,
    SUM(cell_paid)                              AS paid_dollars,
    ROUND(100.0 * SUM(cell_paid)
          / SUM(SUM(cell_paid)) OVER (), 1)     AS pct_of_dollars
FROM cell
GROUP BY 1;

-- PASS C - providers with no usable specialty: no reference row, or a NULL specialty.
-- In most warehouses this is a larger group than anyone expects, and every one of them is
-- invisible to every peer test in this family.
SELECT
    cl.rendering_npi,
    CASE WHEN pr.npi IS NULL THEN 'NO_REFERENCE_ROW' ELSE 'NULL_SPECIALTY' END AS specialty_gap,
    COUNT(*)                                    AS line_count,
    SUM(cl.paid_amount)                         AS paid_dollars
FROM claim_lines cl
LEFT JOIN provider_reference pr ON pr.npi = cl.rendering_npi
WHERE cl.service_date BETWEEN :period_start AND :period_end
  AND cl.adjudication_status = 'PAID'
  AND (pr.npi IS NULL OR pr.specialty_taxonomy IS NULL)
GROUP BY cl.rendering_npi,
         CASE WHEN pr.npi IS NULL THEN 'NO_REFERENCE_ROW' ELSE 'NULL_SPECIALTY' END
ORDER BY paid_dollars DESC;

-- Pass C does not check that a populated specialty is a valid code. If you hold a
-- taxonomy code list, add a third gap value for codes that are not on it.
```

**4 · Threshold guidance.** :mix_divergence_cut = 0.60 on total variation distance is a deliberately loose start - it flags
providers whose billed mix shares less than about 40% of its mass with the other providers in
its recorded specialty (the provider's own billing is left out of the reference mix).
Legitimate sub-specialisation will trip it, so treat pass A output as a data-quality worklist,
not an integrity finding. :min_mix_providers = 5 is the smallest specialty in which pass A
scores anyone; below it the providers are listed as too thin to compare, because a reference
mix built from one or two other practices cannot say which practice is the odd one. It is a
judgement, not a rule. :min_peer_cell = 20 is a judgement, not a rule; the honest way to set it
is to compute, per cell size, how much a single provider's own value moves its peer mean, and
set the floor where that leverage becomes acceptable to you. If pass B shows more than a
quarter of dollars in uncomparable cells, fix the peer construction - broaden the region,
collapse taxonomy to a parent level - before running test 1 at all. Tune to your population; a
starting point, not a standard.

**5 · Paired control + failure mode.** **PE-02 - PECOS/enrollment-of-record data reconciliation cycle.** The specialty taxonomy that defines every peer group in this family lives on the enrollment record, and PE-02 is the control that keeps that record accurate. A large pass-A list is a PE-02 finding, not a provider finding. MD-16 (T-MSIS submission and data-quality validation) is the Medicaid analogue where provider attributes flow to CMS; PI-01 is the downstream consumer that silently inherits the error.

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No reconciliation cycle exists between the enrollment record and any authoritative source, so specialty is whatever was typed at enrollment and never revisited. Expect divergence to correlate with enrollment age. |
| **Designed-not-operating** | A reconciliation cycle is documented at a quarterly cadence but the last exception report predates the period under review. Check the report date first. |
| **Operated-but-bypassed** | Reconciliation covers new enrollments only, so the long-tenured providers - the ones with the largest paid totals - are exactly the ones never re-verified. |
| **Operated-but-ineffective** | The cycle checks that a taxonomy value is present and syntactically valid, not that it is right. Presence checks pass while the peer groups stay wrong; pass A is the test that distinguishes them. |
| **Evidence-missing** | Exceptions were identified but there is no record of resolution - no updated record, no provider contact, no closure date - so the same divergences reappear next cycle. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 3. Outlier decomposition - is it price, volume, or panel?

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
| **Secondary controls** | PI-01 - Routine claims-aberrancy analytics (peer-outlier / statistical); CB-19 - Coding audit program (statistically valid sampling) + error-rate … |

**1 · Data hypothesis.** A cost-per-beneficiary outlier is an identity with three factors: dollars per beneficiary
equals services per beneficiary times dollars per service, over a panel of a given size. Those
three factors point at three completely different control failures. High services per
beneficiary with normal price is a utilisation or medical-necessity question. Normal services
with high price per service is a coding question - upcoding, modifier abuse, place-of-service.
A tiny panel with enormous per-head spend is usually either a genuine complex-care practice or
a phantom-panel scheme. If the risk-scoring and prioritisation model ranks outliers by z-score
alone and never decomposes them, every flag arrives at the reviewer's desk with no hypothesis
attached, and review capacity is spent rediscovering the same three distinctions by hand.

**2 · What to check.** For each flagged provider from test 1, compute the three factors and express each as its own
peer ratio. Assign a driver label to whichever factor carries the largest share of the
deviation. Route the flag by label: price-driven flags to the coding family (family 2), volume-
driven flags to utilisation review, panel-driven flags to eligibility and identity checks
(family 4).

**3 · Reference SQL.**

```sql
-- TEST 3 - Decompose the outlier so the flag names its own driver
-- Runs on the output population of TEST 1. Review candidates, not dispositions.
-- Params: :period_start, :period_end, :min_panel_size (30)

WITH prov AS (
  SELECT
      cl.rendering_npi,
      pr.specialty_taxonomy,
      pr.region_code,
      COUNT(DISTINCT cl.beneficiary_id)                     AS panel_size,
      COUNT(*)                                              AS service_count,
      SUM(cl.paid_amount)                                   AS paid_total,
      SUM(cl.paid_amount) / NULLIF(COUNT(*), 0)             AS price_per_service,
      COUNT(*) * 1.0 / NULLIF(COUNT(DISTINCT cl.beneficiary_id), 0) AS services_per_bene
  FROM claim_lines cl
  JOIN provider_reference pr ON pr.npi = cl.rendering_npi
  WHERE cl.service_date BETWEEN :period_start AND :period_end
    AND cl.adjudication_status = 'PAID'
  GROUP BY cl.rendering_npi, pr.specialty_taxonomy, pr.region_code
  HAVING COUNT(DISTINCT cl.beneficiary_id) >= :min_panel_size
),
peer AS (
  SELECT specialty_taxonomy, region_code,
         AVG(price_per_service)  AS peer_price,
         AVG(services_per_bene)  AS peer_svc,
         AVG(panel_size)         AS peer_panel
  FROM prov
  GROUP BY specialty_taxonomy, region_code
),
ratios AS (
  SELECT
      p.rendering_npi, p.specialty_taxonomy, p.panel_size, p.paid_total,
      p.price_per_service / NULLIF(pe.peer_price, 0) AS price_r,
      p.services_per_bene / NULLIF(pe.peer_svc,   0) AS volume_r,
      p.panel_size        / NULLIF(pe.peer_panel, 0) AS panel_r
  FROM prov p
  JOIN peer pe
    ON pe.specialty_taxonomy = p.specialty_taxonomy
   AND pe.region_code        = p.region_code
)
SELECT
    p.rendering_npi,
    p.specialty_taxonomy,
    p.panel_size,
    p.paid_total,
    ROUND(p.price_r,  2)                                     AS price_ratio,
    ROUND(p.volume_r, 2)                                     AS volume_ratio,
    ROUND(p.panel_r,  2)                                     AS panel_ratio,
    -- the driver is the factor with the largest peer ratio; a tie is reported as MIXED
    CASE
      WHEN p.price_r  > p.volume_r AND p.price_r  > p.panel_r  THEN 'PRICE_DRIVEN'
      WHEN p.volume_r > p.price_r  AND p.volume_r > p.panel_r  THEN 'VOLUME_DRIVEN'
      WHEN p.panel_r  > p.price_r  AND p.panel_r  > p.volume_r THEN 'PANEL_DRIVEN'
      ELSE 'MIXED'
    END                                                      AS primary_driver,
    CASE WHEN p.panel_r < 0.5
              AND p.paid_total / NULLIF(p.panel_size, 0)
                > 2 * (SELECT AVG(paid_total / NULLIF(panel_size,0)) FROM prov)
         THEN 'SMALL_PANEL_HIGH_SPEND'                       -- look at family 4 before family 2
    END                                                      AS panel_flag
FROM ratios p
ORDER BY p.paid_total DESC;

-- The driver label is a routing hint, not a conclusion. A provider can be high on both
-- factors, and the interesting cases usually are. Keep both ratios on the record.
```

**4 · Threshold guidance.** There is no threshold in this test and that is deliberate - it runs on a population already
selected by test 1 and its job is to explain, not to select. The one number worth tuning is the
small-panel rule (panel below half the peer average with per-head spend above twice the
population average); set it from your own distribution, and expect legitimate complex-care and
transplant practices to sit in it permanently, which is a reason to maintain a documented
exclusion list rather than to soften the rule. Tune to your population; a starting point, not a
standard.

**5 · Paired control + failure mode.** **PI-02 - provider risk-scoring and review-prioritisation model.** Decomposition is what turns a ranking into a prioritised queue: the model's job is not to say who is highest but to say who is highest *and why*, so review capacity goes to the right reviewer. A prioritisation model that emits an undifferentiated ranked list is operating but not doing its work. CB-19 (coding audit program) is where price-driven flags should land.

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No prioritisation model exists; outliers are worked in whatever order they were produced, or by dollars alone, so cheap-to-resolve coding issues and expensive investigations compete for the same queue. |
| **Designed-not-operating** | A risk-scoring model is documented but the score is not attached to the flags reviewers actually receive - a common gap where the model lives in an analytics environment and the queue lives in a case system. |
| **Operated-but-bypassed** | Scores are computed but review order is set by referral source or by who complained, and the score is reconstructed afterwards to justify the order. Compare score rank against actual work-start sequence. |
| **Operated-but-ineffective** | The model ranks on a single composite with no driver decomposition, so the queue is ordered but uninformative. If reviewers routinely re-triage flags on receipt, the model is not doing the work. |
| **Evidence-missing** | The score exists at the moment of triage but is not persisted with the case, so no one can later reconstruct why this provider was reviewed and that one was not - which is the record you need when the selection itself is challenged. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 4. Estimator fragility - parametric z-score versus robust median/MAD

| | |
|---|---|
| **Paired control** | **AG-01** - AI governance to NIST AI RMF + Generative AI Profile - maintain a model inventory with intended-use statements, pre-deployment testing, a human-in-the-loop gate before any adverse action, and incident disclosure for any AI/analytics used in coverage, adjudication or fraud detection (NIST guidance; sampled OIG corporate integrity agreements require only a statement on generative-AI use). See Module 08 assessment (G1 adversary / G2 adjudication batteries) |
| **Control type** | Preventive |
| **Module** | Module 08 - AI & Analytics Governance |
| **Owner** |  |
| **Frequency** |  |
| **System touchpoint** |  |
| **Evidence artifact** |  |
| **Control failure mode** |  |
| **Regulatory anchor** | NIST AI RMF 1.0 (AI 100-1): GOVERN 1.6, MAP 3.5, MEASURE 2.1 and 2.3, MANAGE 4.3 [guidance]; NIST AI 600-1 Generative AI Profile: GV-1.6-001 to -003 [guidance, voluntary]; OIG corporate integrity agreements from 2026 (sampled) require a statement on generative-AI use in annual and independent-review reports and do not prescribe a model inventory, human gate or incident disclosure [case evidence] |
| **Secondary controls** | PI-01 - Routine claims-aberrancy analytics (peer-outlier / statistical); PI-02 - Provider risk-scoring & review-prioritization model |

**1 · Data hypothesis.** A z-score built on the mean and standard deviation of a peer cell is computed from statistics
that the outliers themselves contaminate. In a small or heavily skewed cell, a handful of
extreme providers inflate both the mean and the standard deviation, pulling the cut-off up
behind them - so the worst actors raise the bar they have to clear, and the moderately aberrant
providers just below them disappear entirely. This is masking, and it is not a hypothetical: it
is the ordinary behaviour of a parametric estimator on a right-skewed distribution. If the
analytics governance regime never requires an estimator to be validated before deployment, this
failure is invisible, because the model produces a plausible-looking list either way.

**2 · What to check.** Compute the same outlier ranking twice - once with mean and standard deviation, once with
median and median absolute deviation - and compare the two lists. Report providers that appear
in the robust list but not the parametric one; those are the candidates the conventional test
masked. Report the peer cells where the two rankings disagree most, because those are the cells
where the parametric assumption is failing and every future run on that cell is suspect.

**3 · Reference SQL.**

```sql
-- TEST 4 - Estimator fragility: does a robust statistic find providers the z-score misses?
-- A model-validation test, not a provider test. Its output is a list of MODEL findings.
-- Params: :period_start, :period_end, :min_panel_size (30), :z_cut (2.5), :robust_cut (3.5)
-- Portability: PERCENTILE_CONT(0.5) WITHIN GROUP is the SQL-standard median; where it is missing use MEDIAN() or an approximate quantile function. STDDEV is STDDEV_SAMP.
-- Cost here is RAW cost per beneficiary with no risk adjustment: the output compares two
-- estimators on the same input. It is not a provider outlier list (test 1 is).

WITH prov AS (
  SELECT cl.rendering_npi, pr.specialty_taxonomy, pr.region_code,
         SUM(cl.paid_amount) / NULLIF(COUNT(DISTINCT cl.beneficiary_id), 0) AS cost_per_bene
  FROM claim_lines cl
  JOIN provider_reference pr ON pr.npi = cl.rendering_npi
  WHERE cl.service_date BETWEEN :period_start AND :period_end
    AND cl.adjudication_status = 'PAID'
  GROUP BY cl.rendering_npi, pr.specialty_taxonomy, pr.region_code
  HAVING COUNT(DISTINCT cl.beneficiary_id) >= :min_panel_size
),
stats AS (
  SELECT
      specialty_taxonomy, region_code,
      AVG(cost_per_bene)                                                   AS peer_mean,
      STDDEV(cost_per_bene)                                                AS peer_sd,
      PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY cost_per_bene)           AS peer_median
  FROM prov
  GROUP BY specialty_taxonomy, region_code
),
mad AS (
  -- median absolute deviation: the robust analogue of the standard deviation. The mean
  -- absolute deviation is carried as the fallback for cells where more than half the
  -- providers share one cost, which makes the MAD exactly zero.
  SELECT p.specialty_taxonomy, p.region_code,
         PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY ABS(p.cost_per_bene - s.peer_median))
           AS peer_mad,
         AVG(ABS(p.cost_per_bene - s.peer_median)) AS peer_meanad
  FROM prov p
  JOIN stats s ON s.specialty_taxonomy = p.specialty_taxonomy
              AND s.region_code        = p.region_code
  GROUP BY p.specialty_taxonomy, p.region_code
),
scored AS (
  SELECT
      p.rendering_npi, p.specialty_taxonomy, p.region_code, p.cost_per_bene,
      (p.cost_per_bene - s.peer_mean) / NULLIF(s.peer_sd, 0)               AS z_par,
      -- 0.6745 rescales MAD so the robust score is on the same footing as a z-score; when
      -- MAD = 0 the score uses 1.253314 * mean absolute deviation (same scale for a normal
      -- cell); if that is also 0 the robust score is undefined and is reported as such.
      -- The fallback is DISCONTINUOUS: the mean absolute deviation is far larger than a
      -- near-zero MAD, so the same provider scores much lower in a perfectly uniform cell
      -- than in an almost uniform one. Read scores from zero-MAD cells as a floor.
      CASE
        WHEN m.peer_mad > 0
          THEN 0.6745 * (p.cost_per_bene - s.peer_median) / m.peer_mad
        WHEN m.peer_meanad > 0
          THEN (p.cost_per_bene - s.peer_median) / (1.253314 * m.peer_meanad)
      END                                                                  AS z_rob
  FROM prov p
  JOIN stats s ON s.specialty_taxonomy = p.specialty_taxonomy AND s.region_code = p.region_code
  JOIN mad   m ON m.specialty_taxonomy = p.specialty_taxonomy AND m.region_code = p.region_code
)
SELECT
    rendering_npi,
    specialty_taxonomy,
    region_code,
    ROUND(cost_per_bene, 2)                                                AS cost_per_bene,
    ROUND(z_par, 2)                                                        AS z_parametric,
    ROUND(z_rob, 2)                                                        AS z_robust,
    CASE
      WHEN z_rob > :robust_cut AND z_par <= :z_cut
        THEN 'MASKED_BY_PARAMETRIC'          -- the parametric test missed this provider
      WHEN z_par > :z_cut AND z_rob <= :robust_cut
        THEN 'PARAMETRIC_ONLY'               -- likely a skew artifact, not an outlier
      WHEN z_par > :z_cut AND z_rob IS NULL
        THEN 'ROBUST_UNDEFINED'              -- no spread in the cell to scale by
      ELSE 'AGREE'
    END                                                                    AS estimator_agreement
FROM scored
WHERE z_par > :z_cut
   OR z_rob > :robust_cut
ORDER BY estimator_agreement, z_robust DESC;

-- Cell-level companion: rank peer cells by disagreement rate. A cell where the two
-- estimators disagree on most of the tail should not be scored parametrically at all.
```

**4 · Threshold guidance.** :robust_cut = 3.5 on the rescaled robust score is the standard convention and is roughly
comparable in strictness to a parametric z of 2.5-3.0 on well-behaved data; the pairing matters
more than either number, because the point of the test is the disagreement, not the ranking.
Note honestly that neither estimator is right in general - the robust version resists
contamination but is less sensitive in genuinely normal cells. The defensible position is to
compute both, publish the union for triage, and record which estimator produced each flag. One
known limit: when more than half a cell shares exactly one cost the MAD is zero and the score
falls back to the mean absolute deviation, and that fallback is discontinuous. On the seeded
data a provider 20 percent above a perfectly uniform cell scores about 2.7, under the cut,
while the same gap above a cell that is only nearly uniform scores between 15 and 46. Treat a
score from a zero-MAD cell as a lower bound and review such cells by hand. The test also runs
on raw cost per beneficiary with no risk adjustment, so its rows are estimator disagreements to
investigate, not provider outliers. Tune to your population; a starting point, not a standard.

**5 · Paired control + failure mode.** **AG-01 - AI governance to NIST AI RMF, including model inventory, intended-use statements, pre-deployment testing and a human-in-the-loop gate before adverse action.** A peer-group outlier model is a model, whether or not anyone calls it AI, and estimator choice is a pre-deployment testing question. This is the first test in the pack paired to AG-01, and it is the natural one: the finding is about the detection model, not about any provider. Nothing here should reach a provider.

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No model inventory exists, so the outlier statistic is not registered as a model at all and no one owns its assumptions. The estimator was chosen once, by whoever wrote the first query, and has never been revisited. |
| **Designed-not-operating** | An inventory and intended-use statement exist on paper but the outlier model is not in them - analytics built inside the PI unit routinely escape a governance regime aimed at procured AI systems. |
| **Operated-but-bypassed** | The model is registered and governed, but production runs use a locally modified threshold or a different peer definition than the registered version. Compare the parameters in the registered spec against the parameters in the last production run. |
| **Operated-but-ineffective** | Pre-deployment testing was performed but only for reproducibility and runtime, not for estimator validity or masking. A model can pass every engineering test and still be measuring the wrong thing. |
| **Evidence-missing** | No record of which model version and which thresholds produced a given historical flag list, so past selections cannot be reconstructed or defended - the single most common analytics-governance gap and the hardest to remediate after the fact. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 5. Pharmacy cost-per-beneficiary outlier and high-cost drug concentration

| | |
|---|---|
| **Paired control** | **MC-10** - Routine claims & PDE analytics for aberrant provider/pharmacy billing |
| **Control type** | Detective |
| **Module** | Module 06 - Managed Care (Part C/D) |
| **Owner** | PI data analyst |
| **Frequency** | Monthly |
| **System touchpoint** | Claims/PDE analytics |
| **Evidence artifact** | Aberrancy alert disposition |
| **Control failure mode** | Aberrant billing undetected |
| **Regulatory anchor** | Prescription Drug Benefit Manual Pub 100-18 Ch. 9 (FWA) [guidance]; Program Integrity Alliance - Value-for-Money / GovQuery (PIA Connect) [case evidence] |
| **Secondary controls** | MC-13 - Pharmacy audits (desk & on-site) for non-dispensed / phantom Rx; PI-01 - Routine claims-aberrancy analytics (peer-outlier / statistical) |

**1 · Data hypothesis.** Pharmacy is the surface where a peer-cost outlier is most informative, because the unit of
production is standardised: a drug is the same drug everywhere, so a pharmacy whose cost per
beneficiary is far above its peers is either dispensing a different mix or dispensing more of
it. Where routine pharmacy and PDE analytics are absent, the recurring patterns - non-dispensed
prescriptions, a small pharmacy with an improbable concentration in one high-cost specialty
drug, an independent whose beneficiaries mostly live nowhere near it - all show up in the same
aggregate first. Concentration is the discriminating column: legitimate high-cost pharmacies
are high cost across a broad mix; scheme pharmacies are high cost because of one or two codes.

**2 · What to check.** For each pharmacy (service-provider NPI), compute paid dollars per distinct beneficiary,
prescriptions per beneficiary, and the share of paid dollars accounted for by its single
largest drug. Compare against peers grouped by pharmacy type - chain, independent, mail-order,
long-term-care, specialty - because those groups have genuinely different cost structures and
pooling them makes the test meaningless. Report cost z-score and top-drug concentration side by
side.

**3 · Reference SQL.**

```sql
-- TEST 5 - Pharmacy cost-per-beneficiary outlier with concentration context
-- Pattern only. Review candidates, not dispositions. No PHI leaves your environment.
-- Params: :period_start, :period_end, :min_bene (25), :z_cut (2.5), :conc_cut (0.40)
-- Portability: STDDEV here is the sample standard deviation; the standard SQL name is STDDEV_SAMP.

WITH rx AS (
  SELECT
      p.service_provider_npi                          AS pharmacy_npi,
      ph.pharmacy_type,                               -- chain / independent / mail / LTC / specialty
      ph.region_code,
      COUNT(DISTINCT p.beneficiary_id)                AS bene_count,
      COUNT(*)                                        AS fill_count,
      SUM(p.total_paid_amount)                        AS paid_total
  FROM pharmacy_claims p                              -- PDE / NCPDP-derived table in your warehouse
  JOIN pharmacy_reference ph ON ph.npi = p.service_provider_npi
  WHERE p.date_of_service BETWEEN :period_start AND :period_end
    AND p.adjudication_status = 'PAID'
  GROUP BY p.service_provider_npi, ph.pharmacy_type, ph.region_code
  HAVING COUNT(DISTINCT p.beneficiary_id) >= :min_bene
),
top_drug AS (
  SELECT pharmacy_npi, ndc_or_drug_name, drug_paid,
         drug_paid / SUM(drug_paid) OVER (PARTITION BY pharmacy_npi) AS drug_share,
         ROW_NUMBER() OVER (PARTITION BY pharmacy_npi ORDER BY drug_paid DESC) AS rn
  FROM (
    SELECT p.service_provider_npi AS pharmacy_npi,
           p.ndc_or_drug_name,
           SUM(p.total_paid_amount) AS drug_paid
    FROM pharmacy_claims p
    WHERE p.date_of_service BETWEEN :period_start AND :period_end
      AND p.adjudication_status = 'PAID'
    GROUP BY p.service_provider_npi, p.ndc_or_drug_name
  ) d
),
peer AS (
  SELECT pharmacy_type, region_code,
         AVG(paid_total / NULLIF(bene_count, 0))    AS peer_mean,
         STDDEV(paid_total / NULLIF(bene_count, 0)) AS peer_sd
  FROM rx
  GROUP BY pharmacy_type, region_code
)
SELECT
    r.pharmacy_npi,
    r.pharmacy_type,
    r.bene_count,
    r.fill_count,
    ROUND(r.fill_count * 1.0 / NULLIF(r.bene_count, 0), 2)                 AS fills_per_bene,
    r.paid_total,
    ROUND(r.paid_total / NULLIF(r.bene_count, 0), 2)                       AS cost_per_bene,
    ROUND(((r.paid_total / NULLIF(r.bene_count, 0)) - pe.peer_mean)
          / NULLIF(pe.peer_sd, 0), 2)                                      AS z_score,
    td.ndc_or_drug_name                                                    AS top_drug,
    ROUND(td.drug_share, 3)                                                AS top_drug_share,
    CASE WHEN td.drug_share > :conc_cut THEN 'CONCENTRATED' ELSE 'BROAD_MIX' END AS mix_profile
FROM rx r
JOIN peer pe ON pe.pharmacy_type = r.pharmacy_type AND pe.region_code = r.region_code
LEFT JOIN top_drug td ON td.pharmacy_npi = r.pharmacy_npi AND td.rn = 1
WHERE ((r.paid_total / NULLIF(r.bene_count, 0)) - pe.peer_mean)
      / NULLIF(pe.peer_sd, 0) > :z_cut
ORDER BY z_score DESC, r.paid_total DESC;

-- Companion pass worth running alongside: distance between the beneficiary's residence
-- ZIP and the pharmacy ZIP, aggregated to a median per pharmacy. Use ZIP-level centroids
-- only - never beneficiary addresses - and report the median, never the rows.
```

**4 · Threshold guidance.** :conc_cut = 0.40 (a single drug carrying more than 40% of a pharmacy's paid dollars) is a
starting point that will legitimately capture specialty and infusion pharmacies, which is why
pharmacy_type must be in the peer key rather than in the threshold. :min_bene = 25 excludes the
tiny-denominator noise that dominates any pharmacy ranking. The most useful tuning move is to
run the test with and without the highest-cost therapeutic class excluded: if the outlier list
is unchanged, the signal is structural rather than drug-specific. Tune to your population; a
starting point, not a standard.

**5 · Paired control + failure mode.** **MC-10 - routine claims and PDE analytics for aberrant provider/pharmacy billing.** This test is MC-10 executing on the pharmacy side. MC-13 (desk and on-site pharmacy audits for non-dispensed or phantom prescriptions) is the follow-on control that converts a flag into evidence, and the pairing matters: an analytics control with no audit capability behind it generates lists no one can resolve.

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No routine pharmacy analytics run; pharmacy oversight is complaint-driven or delegated entirely to the PBM with no payer-side view. Ask who last looked at a pharmacy-level cost distribution and when. |
| **Designed-not-operating** | Analytics are contractually required of the PBM or delegated entity and are not being produced, or are produced and not shared with the plan. The plan retains the control even where the work is delegated. |
| **Operated-but-bypassed** | Analytics exclude the pharmacies with the strongest business relationship - chain contracts, an affiliated mail-order or specialty pharmacy - which is precisely the population where a payer-side view adds most. |
| **Operated-but-ineffective** | The analytics pool all pharmacy types into one peer group, so specialty and infusion pharmacies permanently occupy the top of the list and crowd out everything else. If your outlier list is the same every quarter, the peer key is wrong. |
| **Evidence-missing** | Flags were generated but the audit trail stops there - no record of which flags went to MC-13 audits, what those audits found, or why the rest were closed. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 6. Beneficiary-side utilisation outlier and cross-provider concentration

| | |
|---|---|
| **Paired control** | **MC-14** - Opioid overutilization monitoring & drug-management program (DMP) |
| **Control type** | Detective |
| **Module** | Module 06 - Managed Care (Part C/D) |
| **Owner** | Pharmacy / clinical lead |
| **Frequency** | Continuous |
| **System touchpoint** | DUR / DMP system |
| **Evidence artifact** | DMP case & POS-edit log |
| **Control failure mode** | Opioid overutilization / doctor-shopping |
| **Regulatory anchor** | 42 CFR 423.153(f) [binding] |
| **Secondary controls** | PI-21 - Beneficiary-side integrity analytics - detect recruiter-clustered …; MD-14 - Utilization control & SURS medical-review program |

**1 · Data hypothesis.** Every test so far ranks providers. The same arithmetic run from the member side finds a
different and complementary population: beneficiaries whose utilisation is far above risk-
matched peers, or who are receiving the same service category from an implausible number of
unrelated providers. That pattern has two very different explanations and the data cannot
distinguish them - it is the signature of a beneficiary being used as a billing vehicle by
multiple providers, and it is equally the signature of a genuinely complex patient with
fragmented care and no case management. Where overutilisation monitoring is absent, neither one
is seen, and the second is the more common. Treating this test as a fraud test rather than a
care-coordination test is the standing error in this space.

**2 · What to check.** For each beneficiary, count distinct billing entities per service category, total paid dollars,
and services per month enrolled; compare against peers matched on risk score band and
eligibility category. Return the tail, but route it by shape: many providers with low per-
provider volume points at fragmentation or at recruitment; few providers with very high volume
points at the provider side and belongs back in test 1.

**3 · Reference SQL.**

```sql
-- TEST 6 - Beneficiary-side utilisation outlier: read this as a care question first
-- Output is a REVIEW population for care management and integrity review jointly.
-- Never route beneficiary-level output to a payment or eligibility action.
-- Params: :period_start, :period_end, :min_months (6), :z_cut (3.0), :provider_cut (8)
-- Portability: STDDEV here is the sample standard deviation; the standard SQL name is STDDEV_SAMP.

WITH bene AS (
  SELECT
      cl.beneficiary_id,
      br.risk_band,                                   -- decile or band of your existing risk score
      br.eligibility_category,
      br.months_enrolled,
      COUNT(DISTINCT cl.billing_npi)                          AS distinct_billers,
      COUNT(DISTINCT cl.service_category)                     AS distinct_categories,
      COUNT(*)                                                AS service_count,
      SUM(cl.paid_amount)                                     AS paid_total,
      SUM(cl.paid_amount) / NULLIF(br.months_enrolled, 0)     AS paid_per_month
  FROM claim_lines cl
  JOIN beneficiary_risk br ON br.beneficiary_id = cl.beneficiary_id
                          AND br.risk_year = EXTRACT(YEAR FROM cl.service_date)
  WHERE cl.service_date BETWEEN :period_start AND :period_end
    AND cl.adjudication_status = 'PAID'
    AND br.months_enrolled >= :min_months
  GROUP BY cl.beneficiary_id, br.risk_band, br.eligibility_category, br.months_enrolled
),
peer AS (
  SELECT risk_band, eligibility_category,
         AVG(paid_per_month)    AS peer_mean,
         STDDEV(paid_per_month) AS peer_sd,
         AVG(distinct_billers)  AS peer_billers
  FROM bene
  GROUP BY risk_band, eligibility_category
)
SELECT
    b.beneficiary_id,                                  -- pseudonymise before ANY export
    b.risk_band,
    b.eligibility_category,
    b.months_enrolled,
    b.distinct_billers,
    b.distinct_categories,
    b.service_count,
    b.paid_total,
    ROUND((b.paid_per_month - p.peer_mean) / NULLIF(p.peer_sd, 0), 2) AS z_score,
    ROUND(b.distinct_billers / NULLIF(p.peer_billers, 0), 2)          AS biller_ratio,
    CASE
      WHEN b.distinct_billers >= :provider_cut
       AND b.paid_total / NULLIF(b.distinct_billers, 0)
           < (SELECT AVG(paid_total) FROM bene)
        THEN 'FRAGMENTED_MANY_LOW'      -- care-coordination or recruitment pattern
      ELSE 'CONCENTRATED_FEW_HIGH'      -- send back to the provider-side tests
    END                                                               AS shape
FROM bene b
JOIN peer p ON p.risk_band = b.risk_band
           AND p.eligibility_category = b.eligibility_category
WHERE (b.paid_per_month - p.peer_mean) / NULLIF(p.peer_sd, 0) > :z_cut
   OR b.distinct_billers >= :provider_cut
ORDER BY z_score DESC;

-- Aggregate before you report. The reportable artifact from this test is counts and rates
-- by shape and risk band, plus a referral list to care management - not a row dump.
```

**4 · Threshold guidance.** :z_cut = 3.0 is deliberately tighter than the provider-side 2.5, because beneficiary cost
distributions are far more skewed - a small number of members legitimately account for a large
share of spend, and flagging them adds nothing anyone did not already know. :provider_cut = 8
distinct billing entities in a period is a starting point only and varies enormously by
eligibility category; dual-eligible and LTSS populations will exceed it routinely and should be
banded separately rather than excluded. Tune to your population; a starting point, not a
standard.

**5 · Paired control + failure mode.** **MC-14 - opioid overutilisation monitoring and drug-management programme.** MC-14 is the library's only beneficiary-level overutilisation control, and it is drug-specific; this test is its generalisation to service utilisation, which is why gap note G3 recommends a broader control. PI-21 (beneficiary-side integrity analytics) covers the recruitment interpretation; MD-14 (SURS) is the Medicaid utilisation-control hook. The beneficiary-protection framing of MC-14 - the member is a subject of care, not a suspect - should carry over to any use of this test.

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No beneficiary-level utilisation monitoring of any kind; all surveillance is provider-side, so members being used as billing vehicles across several providers are invisible because no single provider looks unusual. |
| **Designed-not-operating** | A drug-management or overutilisation programme exists on paper with case criteria but no case volume - check whether any beneficiary was actually reviewed in the period. |
| **Operated-but-bypassed** | Monitoring covers only the drug dimension MC-14 names and stops at the boundary of the pharmacy benefit, so service-side overutilisation has no owner at all. |
| **Operated-but-ineffective** | Flags are generated but routed exclusively to integrity review, so members with fragmented care are investigated rather than case-managed. This is a real harm, not merely an inefficiency, and it is the failure mode to look for first. |
| **Evidence-missing** | No record of the disposition split between care-management referral and integrity referral, so the programme cannot show it is not simply investigating sick people. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 7. Institutional cost-per-day outlier and cost-report-to-claims divergence

| | |
|---|---|
| **Paired control** | **MD-23** - Nursing-facility cost-report rate-setting integrity check - cross-reference self-reported cost data against independent financial-distress indicators (payroll-tax delinquency, vendor/UCC collections, receivership/licensure-action status) before accepting/renewing per-diem rates |
| **Control type** | Detective |
| **Module** | Module 07 - Medicaid & CHIP Integrity |
| **Owner** | State rate-setting / PI unit |
| **Frequency** | Each cost-report cycle |
| **System touchpoint** | Rate-setting system + state revenue / licensure / court data |
| **Evidence artifact** | Financial-distress screen result attached to each accepted cost report |
| **Control failure mode** | Inflated self-reported costs driving overpaid per-diem rates at a financially-distressed facility (pattern from a 2025 state nursing-facility case) |
| **Regulatory anchor** | 42 CFR 447 Subpart C (Payment for Inpatient Hospital & Long-Term Care Facility Services; NF rate reasonableness, SSA §1902(a)(13)(A)) [binding] |
| **Secondary controls** | PI-01 - Routine claims-aberrancy analytics (peer-outlier / statistical); MD-11 - State-directed payment & capitation actuarial-soundness oversight |

**1 · Data hypothesis.** Facilities are paid from rates built on self-reported cost data, and the rate is set once and
then paid on every day of every stay. That makes the facility surface structurally different
from the professional surface: the money does not move because someone billed a wrong code, it
moves because the input to a rate calculation was wrong years earlier and has been compounding
ever since. Where no independent cross-reference of self-reported cost data exists, the
observable signature is divergence - a facility whose claims-derived cost per patient-day
drifts away from its peer group in a direction and at a rate that its case mix does not
explain, and whose reported costs move in the categories that carry the most rate weight.

**2 · What to check.** Compute claims-derived paid dollars per patient-day per facility, adjusted for the acuity mix
the facility itself reports, and compare against peers grouped by facility type, bed size band
and region. Then compare the trajectory: the ratio of the facility's current-period cost per
day to its own prior-period cost per day, against the same ratio for its peer group. A facility
moving away from its peers over time is a stronger candidate than one that has simply always
been expensive.

**3 · Reference SQL.**

```sql
-- TEST 7 - Institutional cost-per-day outlier plus trajectory versus peers
-- Pattern only. Review candidates, not dispositions.
-- Params: :period_start, :period_end, :prior_start, :prior_end, :min_days (500), :z_cut (2.5), :growth_cut (0.10)
-- Portability: GROUP BY 5 is an ordinal (repeat the CASE expression where ordinals are not accepted); STDDEV is STDDEV_SAMP; NULLS LAST: sort on (col IS NULL) first.

WITH fac_period AS (
  SELECT
      cl.facility_id,
      f.facility_type,                                 -- SNF / NF / ICF / IRF / LTACH / hospice
      f.bed_size_band,
      f.region_code,
      CASE WHEN cl.service_date BETWEEN :period_start AND :period_end THEN 'CURRENT'
           WHEN cl.service_date BETWEEN :prior_start  AND :prior_end  THEN 'PRIOR'
      END                                              AS period_label,
      SUM(cl.covered_days)                             AS patient_days,
      COUNT(DISTINCT cl.beneficiary_id)                AS residents,
      SUM(cl.paid_amount)                              AS paid_total,
      AVG(cl.case_mix_index)                           AS mean_cmi   -- as reported by the facility
  FROM institutional_claims cl
  JOIN facility_reference f ON f.facility_id = cl.facility_id
  WHERE cl.service_date BETWEEN :prior_start AND :period_end
    AND cl.adjudication_status = 'PAID'
  GROUP BY cl.facility_id, f.facility_type, f.bed_size_band, f.region_code, 5
  HAVING SUM(cl.covered_days) >= :min_days
),
pivoted AS (
  SELECT facility_id, facility_type, bed_size_band, region_code,
         MAX(CASE WHEN period_label = 'CURRENT'
                  THEN paid_total / NULLIF(patient_days, 0) / NULLIF(mean_cmi, 0) END) AS cur_cost_day,
         MAX(CASE WHEN period_label = 'PRIOR'
                  THEN paid_total / NULLIF(patient_days, 0) / NULLIF(mean_cmi, 0) END) AS pri_cost_day,
         -- the same cost with NO case-mix adjustment: the pass a facility cannot move by
         -- changing the acuity it reports
         MAX(CASE WHEN period_label = 'CURRENT'
                  THEN paid_total / NULLIF(patient_days, 0) END)                       AS cur_raw_cost_day,
         MAX(CASE WHEN period_label = 'CURRENT' THEN patient_days END)                 AS cur_days,
         MAX(CASE WHEN period_label = 'CURRENT' THEN paid_total   END)                 AS cur_paid,
         MAX(CASE WHEN period_label = 'CURRENT' THEN mean_cmi     END)                 AS cur_cmi
  FROM fac_period
  GROUP BY facility_id, facility_type, bed_size_band, region_code
),
peer AS (
  SELECT facility_type, bed_size_band, region_code,
         AVG(cur_cost_day)                                        AS peer_mean,
         STDDEV(cur_cost_day)                                     AS peer_sd,
         AVG(cur_raw_cost_day)                                    AS peer_raw_mean,
         STDDEV(cur_raw_cost_day)                                 AS peer_raw_sd,
         AVG(cur_cmi)                                             AS peer_cmi,
         AVG(cur_cost_day / NULLIF(pri_cost_day, 0))              AS peer_growth
  FROM pivoted
  GROUP BY facility_type, bed_size_band, region_code
),
scored AS (
  SELECT
      p.facility_id, p.facility_type, p.bed_size_band, p.cur_days, p.cur_paid, p.cur_cmi,
      p.cur_cost_day, p.pri_cost_day, p.cur_raw_cost_day,
      pe.peer_growth,
      pe.peer_cmi,
      (p.cur_cost_day     - pe.peer_mean)     / NULLIF(pe.peer_sd, 0)      AS z_adj,
      (p.cur_raw_cost_day - pe.peer_raw_mean) / NULLIF(pe.peer_raw_sd, 0)  AS z_raw,
      (p.cur_cost_day / NULLIF(p.pri_cost_day, 0)) - pe.peer_growth        AS excess_growth
  FROM pivoted p
  JOIN peer pe ON pe.facility_type   = p.facility_type
              AND pe.bed_size_band   = p.bed_size_band
              AND pe.region_code     = p.region_code
)
SELECT
    facility_id,
    facility_type,
    bed_size_band,
    cur_days,
    cur_paid,
    ROUND(cur_cmi, 3)                                                      AS reported_cmi,
    ROUND(cur_cmi / NULLIF(peer_cmi, 0), 3)                                AS cmi_ratio_to_peer,
    ROUND(cur_cost_day, 2)                                                 AS cmi_adj_cost_per_day,
    ROUND(z_adj, 2)                                                        AS z_score,
    ROUND(cur_raw_cost_day, 2)                                             AS raw_cost_per_day,
    ROUND(z_raw, 2)                                                        AS raw_z_score,
    ROUND(cur_cost_day / NULLIF(pri_cost_day, 0), 3)                       AS own_growth,
    ROUND(peer_growth, 3)                                                  AS peer_growth,
    ROUND(excess_growth, 3)                                                AS excess_growth,
    CASE
      WHEN z_adj > :z_cut AND z_raw > :z_cut THEN 'ADJUSTED_AND_UNADJUSTED'
      WHEN z_adj > :z_cut                    THEN 'CMI_ADJUSTED_ONLY'
      WHEN z_raw > :z_cut                    THEN 'UNADJUSTED_ONLY'   -- reported acuity absorbs the cost
      ELSE 'GROWTH_ONLY'
    END                                                                    AS outlier_basis
FROM scored
WHERE z_adj > :z_cut
   OR z_raw > :z_cut
   OR excess_growth > :growth_cut
ORDER BY excess_growth DESC NULLS LAST, z_score DESC;

-- Note the self-reference: the adjusted cost per day is divided by the case-mix index the
-- facility itself reports, so a facility that inflates its reported acuity LOWERS its own
-- adjusted cost and would hide from an adjusted-only test. The unadjusted pass above is
-- there to catch that. UNADJUSTED_ONLY means the reported case mix accounts for the whole
-- cost difference: that is either genuinely higher acuity or inflated acuity, and claims
-- cannot tell which. Check cmi_ratio_to_peer against clinical indicators before anything
-- else - rising CMI with flat clinical indicators is its own candidate list.
```

**4 · Threshold guidance.** :growth_cut = 0.10 (growing ten percentage points faster than the peer group year over year) is
the number to tune first and the more informative of the two tests; a level outlier can reflect
a legitimately different resident population, but divergence from an appropriate starting point
usually cannot. Adjusting by self-reported case mix is a deliberate compromise recorded in the
SQL comment: it controls for real acuity differences and simultaneously creates a defeat
mechanism, so the query scores both the adjusted and the unadjusted cost and labels which one
produced each row. An UNADJUSTED_ONLY row is a facility whose reported acuity explains its
cost; the query cannot tell real acuity from inflated acuity, so that label is a prompt to
verify the case mix, not a finding. Tune to your population; a starting point, not a standard.

**5 · Paired control + failure mode.** **MD-23 - nursing-facility cost-report rate-setting integrity check, cross-referencing self-reported cost data against independent evidence.** This test is the claims-side half of that cross-reference: it cannot audit a cost report, but it can say which facilities' claims behaviour does not match the rate they are being paid. MD-11 (state-directed payment and capitation actuarial-soundness oversight) is the adjacent control where rate inputs are reviewed.

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | Cost reports are accepted and used for rate-setting with no independent cross-reference of any kind. The rate-setting cycle is annual and the error compounds, so the exposure is a multiple of a single period's divergence. |
| **Designed-not-operating** | A desk-review procedure exists but reviews are not being completed within the rate cycle, so rates are set on unreviewed data and any correction lands too late to affect payment. |
| **Operated-but-bypassed** | Review covers only facilities selected on a rotation or below a size threshold, and the largest facilities - where a rate error is worth the most - are reviewed least often. |
| **Operated-but-ineffective** | Review verifies internal arithmetic and schedule completeness rather than the reasonableness of reported costs against independent evidence, which is the specific thing MD-23 asks for. Related-party transactions are the usual blind spot. |
| **Evidence-missing** | Adjustments were made to reported costs but the basis is not documented, so neither the facility nor a later auditor can reconstruct why the rate is what it is. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 8. Capitation and directed-payment outlier - dollars per member the acuity does not explain

| | |
|---|---|
| **Paired control** | **MD-11** - State-directed payment & capitation actuarial-soundness oversight |
| **Control type** | Preventive |
| **Module** | Module 07 - Medicaid & CHIP Integrity |
| **Owner** | Actuarial / managed care office |
| **Frequency** | Per rate cycle |
| **System touchpoint** | Rate-setting & 438.6 documentation |
| **Evidence artifact** | Actuarial certification + approval file |
| **Control failure mode** | Unsound/unallowable directed payments |
| **Regulatory anchor** | 42 CFR 438.4-.7 (actuarial soundness) [binding]; 438.608 [binding] |
| **Secondary controls** | MD-17 - Managed-care encounter/payment data completeness monitoring + state …; MC-24 - Favorable-selection detection (cherry-picking / lemon-dropping) - … |

**1 · Data hypothesis.** Under capitation the money moves before any service does, so the provider-side tests in this
family are blind to it: a plan or a delegated entity can be paid far above peers per member
with no claim ever looking unusual. Where actuarial-soundness and directed-payment oversight is
weak, the observable signature is a gap between what is being paid per member and what is being
delivered per member - capitation and directed-payment dollars rising while encounter volume
per member stays flat or falls. The same gap has an innocent explanation, incomplete encounter
reporting, and distinguishing the two is the whole difficulty; that is why this test must be
read alongside encounter-completeness monitoring rather than on its own.

**2 · What to check.** For each plan or delegated entity and rating region, compute capitation plus directed-payment
dollars per member month, and encounter-derived service volume and allowed value per member
month. Compare both against peers in the same rating region and product line, and compute the
ratio between them. Flag entities whose payment per member month is high relative to peers
while their delivered value per member month is not - and cross-check every hit against that
entity's encounter-completeness metrics before drawing any conclusion.

**3 · Reference SQL.**

```sql
-- TEST 8 - Capitation / directed-payment per member month versus delivered value
-- Pattern only. Review candidates, not dispositions. Read WITH encounter completeness.
-- Params: :period_start, :period_end, :min_mm (5000), :z_cut (2.0), :gap_cut (1.25)
-- Portability: STDDEV is STDDEV_SAMP in standard SQL; where NULLS LAST is not accepted, sort on (col IS NULL), col DESC.

WITH pay AS (
  SELECT
      c.plan_id,
      c.entity_id,                                     -- delegated entity / sub-cap arrangement
      c.product_line,                                  -- MCO / PIHP / PAHP / D-SNP / etc.
      c.rating_region,
      SUM(c.member_months)                             AS member_months,
      SUM(c.capitation_paid)                           AS cap_paid,
      SUM(c.directed_payment_amount)                   AS directed_paid,
      AVG(c.mean_risk_score)                           AS mean_risk
  FROM capitation_payments c
  WHERE c.payment_month BETWEEN :period_start AND :period_end
  GROUP BY c.plan_id, c.entity_id, c.product_line, c.rating_region
  HAVING SUM(c.member_months) >= :min_mm
),
delivered AS (
  SELECT
      e.plan_id, e.entity_id,
      COUNT(*)                                         AS encounter_count,
      SUM(e.allowed_amount)                            AS encounter_value
  FROM encounter_records e                             -- T-MSIS OT/IP/LT encounter concepts
  WHERE e.service_date BETWEEN :period_start AND :period_end
  GROUP BY e.plan_id, e.entity_id
),
joined AS (
  SELECT
      p.*,
      COALESCE(d.encounter_count, 0)                                   AS encounter_count,
      -- encounters carry no region: allocate the entity's total across its region rows by
      -- member-month share so a multi-region entity is not credited with it in full each time
      COALESCE(d.encounter_value, 0) * p.member_months
        / NULLIF(SUM(p.member_months) OVER (PARTITION BY p.plan_id, p.entity_id), 0)
                                                                       AS encounter_value,
      (p.cap_paid + p.directed_paid) / NULLIF(p.member_months, 0)
        / NULLIF(p.mean_risk, 0)                                       AS adj_pmpm_paid,
      COALESCE(d.encounter_value, 0) / NULLIF(SUM(p.member_months) OVER (PARTITION BY p.plan_id, p.entity_id), 0)
        / NULLIF(p.mean_risk, 0)                                       AS adj_pmpm_delivered
  FROM pay p
  LEFT JOIN delivered d ON d.plan_id = p.plan_id AND d.entity_id = p.entity_id
),
peer AS (
  SELECT product_line, rating_region,
         AVG(adj_pmpm_paid)         AS peer_paid_mean,
         STDDEV(adj_pmpm_paid)      AS peer_paid_sd,
         AVG(adj_pmpm_delivered)    AS peer_delivered_mean
  FROM joined
  GROUP BY product_line, rating_region
)
SELECT
    j.plan_id,
    j.entity_id,
    j.product_line,
    j.rating_region,
    j.member_months,
    ROUND(j.adj_pmpm_paid, 2)                                             AS adj_pmpm_paid,
    ROUND(j.adj_pmpm_delivered, 2)                                        AS adj_pmpm_delivered,
    ROUND((j.adj_pmpm_paid - pe.peer_paid_mean)
          / NULLIF(pe.peer_paid_sd, 0), 2)                                AS paid_z,
    ROUND((j.adj_pmpm_paid / NULLIF(pe.peer_paid_mean, 0))
          / NULLIF(j.adj_pmpm_delivered
                   / NULLIF(pe.peer_delivered_mean, 0), 0), 2)            AS paid_to_delivered_gap,
    j.encounter_count,
    CASE WHEN j.encounter_count = 0 THEN 'NO_ENCOUNTERS_REPORTED'
         WHEN j.adj_pmpm_delivered < 0.5 * pe.peer_delivered_mean
              THEN 'DELIVERY_FAR_BELOW_PEER'
    END                                                                   AS delivery_flag
FROM joined j
JOIN peer pe ON pe.product_line = j.product_line AND pe.rating_region = j.rating_region
WHERE (j.adj_pmpm_paid - pe.peer_paid_mean) / NULLIF(pe.peer_paid_sd, 0) > :z_cut
   OR j.encounter_count = 0                  -- zero delivered value makes the gap ratio NULL
   OR (j.adj_pmpm_paid / NULLIF(pe.peer_paid_mean, 0))
      / NULLIF(j.adj_pmpm_delivered / NULLIF(pe.peer_delivered_mean, 0), 0) > :gap_cut
ORDER BY paid_to_delivered_gap DESC NULLS LAST;

-- MANDATORY companion: pull the same entities' encounter-completeness metrics before
-- interpreting anything. An entity that under-reports encounters looks identical here to
-- an entity that under-delivers care, and the remedy for the two is opposite.
```

**4 · Threshold guidance.** :z_cut = 2.0 is looser than elsewhere in this family because the population is small - there
are tens of plans, not thousands of providers - and a strict cut on a small population finds
nothing. :gap_cut = 1.25 says an entity is paid 25% more relative to peers than it delivers
relative to peers; that is a starting point and is highly sensitive to encounter-data quality,
which is the reason the companion check is marked mandatory rather than optional. Where
encounter completeness is known to be poor, this test should be reported as a data-quality
finding and not as a payment finding at all. Tune to your population; a starting point, not a
standard.

**5 · Paired control + failure mode.** **MD-11 - state-directed payment and capitation actuarial-soundness oversight.** The test asks the question MD-11 owns: is what we are paying per member defensible against what comparable entities are paid and what is being delivered. MD-17 (managed-care encounter/payment data completeness monitoring) is the control that makes the answer interpretable; MC-24 (favourable-selection detection) is the adjacent explanation when the risk mix rather than the rate is doing the work.

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No comparative review of directed payments or capitation against delivered value; soundness is certified on the actuarial submission alone with no retrospective look-back. |
| **Designed-not-operating** | Oversight is defined in the contract or state plan but the retrospective comparison was not performed for the rating period under review. |
| **Operated-but-bypassed** | Review covers base capitation but excludes directed payments and pass-through arrangements, which is where the discretionary dollars and the growth actually are. |
| **Operated-but-ineffective** | Review compares payments against the actuarial certification rather than against delivered value or peers, so an arrangement that was certified on optimistic assumptions is never revisited against what happened. |
| **Evidence-missing** | Encounter data is too incomplete to support any delivered-value comparison, so the oversight cannot be performed at all. Record this as an MD-17 evidence finding, not as an MD-11 payment finding - the distinction is the point. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 9. Documentation-request aging and non-response - flags that never became evidence

| | |
|---|---|
| **Paired control** | **PI-13** - Contractor ADR-response tracking (UPIC/RAC/CERT/MAC) |
| **Control type** | Preventive |
| **Module** | Module 02 - Program Integrity |
| **Owner** | Compliance / HIM |
| **Frequency** | Per ADR |
| **System touchpoint** | ADR tracking system |
| **Evidence artifact** | ADR response + timeliness log |
| **Control failure mode** | Default denials from missed deadlines |
| **Regulatory anchor** | CMS PIM Pub 100-08 Ch.3 (Verifying Errors & Corrective Actions) [guidance]; 42 CFR 405.980 (reopening) / SSA 1893(h) (RAC) [binding] |
| **Secondary controls** | PI-05 - Post-payment medical review w/ valid sampling & extrapolation; CD-14 - Insufficient-documentation root-cause & corrective-action loop from … |

**1 · Data hypothesis.** An outlier flag becomes a finding only if someone asks for records and the records arrive.
Where documentation-request tracking is weak, requests are issued and then silently expire: the
provider does not respond, the clock runs out, and the case is closed as unsubstantiated  - 
which reads in the metrics as a false positive when it is actually an unanswered question. Two
populations are diagnostic. Requests aged past the response deadline with no recorded response
and no recorded escalation, and providers with a persistent non-response rate across multiple
requests. The second is the more serious: a provider who reliably does not answer has learned
that not answering works.

**2 · What to check.** For every documentation request issued in the period, compute days from issue to response or to
today, whether a response was recorded, and whether the case was closed before or after the
deadline. Aggregate to provider level to find persistent non-responders, and to programme level
to find the share of requests that expired without escalation.

**3 · Reference SQL.**

```sql
-- TEST 9 - ADR / documentation-request aging and non-response
-- Runs on your audit-tracking system, not on claims. Two statements: A by programme, B by provider.
-- Params: :period_start, :period_end, :response_days (45), :min_requests (3), :nonresponse_cut (0.30)
-- Portability: DATE_DIFF with a trailing DAY unit is BigQuery syntax (elsewhere DATEDIFF(day, earlier, later) or a date subtraction); for date + integer use DATEADD.

-- STATEMENT A: programme-level aging
WITH req AS (
  SELECT
      r.request_id,
      r.provider_npi,
      r.programme,                                     -- UPIC / RAC / CERT / MAC / internal SIU
      r.issue_date,
      r.response_due_date,
      r.response_received_date,
      r.case_status,
      r.case_closed_date,
      r.dollars_at_risk,
      COALESCE(DATE_DIFF(COALESCE(r.response_received_date, CURRENT_DATE), r.issue_date, DAY),
               0)                                      AS days_open,
      CASE WHEN r.response_received_date IS NULL
            AND COALESCE(r.response_due_date, r.issue_date + :response_days) < CURRENT_DATE
           THEN 1 ELSE 0 END                           AS expired_unanswered
  FROM documentation_requests r
  WHERE r.issue_date BETWEEN :period_start AND :period_end
)
SELECT
    programme,
    COUNT(*)                                                     AS requests,
    SUM(expired_unanswered)                                      AS expired_unanswered,
    ROUND(100.0 * SUM(expired_unanswered) / NULLIF(COUNT(*),0),1) AS pct_expired,
    SUM(CASE WHEN expired_unanswered = 1 THEN dollars_at_risk END) AS dollars_unresolved,
    -- the quiet failure: cases closed as unsubstantiated while the request was still
    -- unanswered. Count these separately; they are not false positives.
    SUM(CASE WHEN case_status = 'CLOSED_UNSUBSTANTIATED'
              AND response_received_date IS NULL THEN 1 ELSE 0 END)
                                                                 AS closed_unsubstantiated_unanswered,
    ROUND(AVG(days_open), 1)                                     AS mean_days_open
FROM req
GROUP BY programme
ORDER BY dollars_unresolved DESC NULLS LAST;

-- STATEMENT B: persistent non-responders - the population that matters most
WITH req AS (
  SELECT
      r.provider_npi,
      r.dollars_at_risk,
      CASE WHEN r.response_received_date IS NULL
            AND COALESCE(r.response_due_date, r.issue_date + :response_days) < CURRENT_DATE
           THEN 1 ELSE 0 END                           AS expired_unanswered
  FROM documentation_requests r
  WHERE r.issue_date BETWEEN :period_start AND :period_end
)
SELECT
    provider_npi,
    COUNT(*)                                             AS requests,
    SUM(expired_unanswered)                              AS unanswered,
    ROUND(1.0 * SUM(expired_unanswered) / COUNT(*), 3)   AS nonresponse_rate,
    SUM(dollars_at_risk)                                 AS exposure
FROM req
GROUP BY provider_npi
HAVING COUNT(*) >= :min_requests
   AND 1.0 * SUM(expired_unanswered) / COUNT(*) > :nonresponse_cut
ORDER BY exposure DESC;
```

**4 · Threshold guidance.** :response_days = 45 reflects a common documentation-request window but yours is set by
programme and by contract - use the actual due date on the record wherever it exists and fall
back to the parameter only where it does not. :nonresponse_cut = 0.30 across at least
:min_requests = 3 requests is a starting point for 'persistent'; the number matters less than
running the provider-level pass at all, since a single unanswered request is routine and a
pattern of them is not. Tune to your population; a starting point, not a standard.

**5 · Paired control + failure mode.** **PI-13 - contractor ADR-response tracking (UPIC/RAC/CERT/MAC).** The closed_unsubstantiated_unanswered column of statement A is the sharpest use of this test: cases closed as unsubstantiated while the request sat unanswered are being counted as analytics false positives, which quietly corrupts every precision metric in test 10 and can lead a programme to tune down a test that was working. PI-05 is the medical-review process the request serves; CD-14 is the insufficient-documentation root-cause loop.

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No central tracking of documentation requests; each reviewer tracks their own, so no one can say how many requests are outstanding or what they are worth. |
| **Designed-not-operating** | A tracking system exists but issue and response dates are not consistently populated, making aging uncomputable - check field completeness before interpreting any aging number. |
| **Operated-but-bypassed** | Requests issued outside the tracked workflow - by email, or by a delegated vendor with its own system - are absent from the tracker entirely, so the measured non-response rate understates the real one. |
| **Operated-but-ineffective** | Aging is tracked but expiry triggers no escalation, so the tracker faithfully records requests dying of old age. Look for the escalation field being uniformly null. |
| **Evidence-missing** | Responses were received but not linked to the request or the case, so the programme cannot demonstrate what evidence it had when it closed a case. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 10. Flag-to-disposition yield - precision, cycle time and return by test

| | |
|---|---|
| **Paired control** | **PI-19** - PI performance reporting (recovery ROI, prevention metrics) |
| **Control type** | Detective |
| **Module** | Module 02 - Program Integrity |
| **Owner** | PI program lead |
| **Frequency** | Quarterly |
| **System touchpoint** | PI dashboard/BI |
| **Evidence artifact** | Governance metrics report |
| **Control failure mode** | No oversight of PI effectiveness |
| **Regulatory anchor** | Program Integrity Alliance - Value-for-Money in Anti-Fraud [case evidence]; GAO Green Book 2025 (Standards for Internal Control) [guidance] |
| **Secondary controls** | PI-02 - Provider risk-scoring & review-prioritization model; AG-01 - AI governance to NIST AI RMF + Generative AI Profile - maintain a … |

**1 · Data hypothesis.** An outlier programme that does not measure its own precision cannot improve and cannot be
defended. The failure has a characteristic shape: flag volume is reported as an accomplishment,
dollars identified are reported as a result, and the two numbers that actually matter - what
share of flags became findings, and what share of identified dollars were recovered - are
absent. Where performance reporting is absent or vanity-metric only, the observable signature
is a programme whose flag volume is stable or growing while its finding rate is unmeasured, and
whose oldest open flags are older than anyone realises.

**2 · What to check.** For each detection test or model that generates flags, compute flags generated, flags triaged,
flags dispositioned, findings confirmed, dollars identified and dollars recovered, plus median
cycle time from flag to disposition and the age of the oldest open flag. Report precision per
test rather than in aggregate, because a single high-precision test can carry a programme
average while every other test runs at near-zero yield.

**3 · Reference SQL.**

```sql
-- TEST 10 - Yield, precision and cycle time by detection test
-- Runs on your PI case-management system. This is the programme's self-measurement.
-- Params: :period_start, :period_end
-- Portability: DATE_DIFF with a trailing DAY unit is BigQuery syntax; elsewhere use DATEDIFF(day, earlier, later) or a date subtraction.

WITH f AS (
  SELECT
      fl.flag_id,
      fl.detection_test_id,                             -- which query/model produced it
      fl.model_version,
      fl.generated_date,
      fl.triage_date,
      fl.disposition,                                   -- e.g. NO_ACTION / EDUCATION / AUDIT / REFERRAL
      fl.disposition_date,
      fl.finding_confirmed,                             -- boolean set at case closure
      fl.dollars_identified,
      fl.dollars_recovered
  FROM pi_flags fl
  WHERE fl.generated_date BETWEEN :period_start AND :period_end
)
SELECT
    detection_test_id,
    model_version,
    COUNT(*)                                                            AS flags_generated,
    SUM(CASE WHEN triage_date IS NOT NULL THEN 1 ELSE 0 END)            AS flags_triaged,
    SUM(CASE WHEN disposition IS NOT NULL THEN 1 ELSE 0 END)            AS flags_dispositioned,
    -- findings and precision count dispositioned flags only, so the numerator is a subset
    -- of the denominator and precision cannot pass 100
    SUM(CASE WHEN finding_confirmed AND disposition IS NOT NULL
             THEN 1 ELSE 0 END)                                         AS findings,
    ROUND(100.0 * SUM(CASE WHEN finding_confirmed AND disposition IS NOT NULL
                           THEN 1 ELSE 0 END)
          / NULLIF(SUM(CASE WHEN disposition IS NOT NULL THEN 1 ELSE 0 END), 0), 1)
                                                                        AS precision_pct,
    -- data-quality column: a confirmed finding with no recorded disposition is a record
    -- error; it is shown here and kept out of precision
    SUM(CASE WHEN finding_confirmed AND disposition IS NULL
             THEN 1 ELSE 0 END)                                         AS confirmed_without_disposition,
    SUM(dollars_identified)                                             AS dollars_identified,
    SUM(dollars_recovered)                                              AS dollars_recovered,
    ROUND(100.0 * SUM(dollars_recovered)
          / NULLIF(SUM(dollars_identified), 0), 1)                      AS recovery_pct,
    -- cycle time on dispositioned flags only, so an untouched backlog cannot flatter it
    ROUND(AVG(CASE WHEN disposition_date IS NOT NULL
                   THEN DATE_DIFF(disposition_date, generated_date, DAY) END), 1)
                                                                        AS mean_days_to_disposition,
    MAX(CASE WHEN disposition IS NULL
             THEN DATE_DIFF(CURRENT_DATE, generated_date, DAY) END)     AS oldest_open_flag_days
FROM f
GROUP BY detection_test_id, model_version
ORDER BY dollars_identified DESC NULLS LAST;

-- Read precision and the untriaged backlog together. A test showing 90% precision on
-- twelve triaged flags out of nine hundred generated is not a precise test; it is an
-- untouched queue with a cherry-picked front. Always report flags_triaged/flags_generated
-- next to precision_pct, and never publish precision without it.
```

**4 · Threshold guidance.** There is no correct precision number and any published target would be misused - a high-
precision programme may simply be running one narrow test and ignoring the rest of its
exposure, and a low-precision one may be doing valuable exploratory work. What is diagnosable
is the *shape*: precision that is unmeasured, precision reported without triage coverage, cycle
times with no ceiling, and an oldest-open-flag age that exceeds the recovery look-back period,
which converts an analytics backlog into an unrecoverable one. Set your own targets from your
own baseline and revisit them when a model version changes. Tune to your population; a starting
point, not a standard.

**5 · Paired control + failure mode.** **PI-19 - PI performance reporting (recovery ROI, prevention metrics).** This test is PI-19 executing at test-level granularity rather than programme level, which is the granularity at which the numbers are actionable. It also feeds AG-01: precision by model version is the model-monitoring metric an analytics governance regime needs, and it cannot be produced unless the flag record carries the model version - which is the single most valuable field to add if it is missing.

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No performance reporting exists; the programme reports activity (flags generated, audits opened) and never outcome. Ask for last year's precision figure and see whether one exists. |
| **Designed-not-operating** | A reporting pack is defined but has not been produced for the period, or is produced for an executive audience only in a form that omits precision and backlog. |
| **Operated-but-bypassed** | Reporting covers the tests that perform well and omits the rest, or resets the denominator each period so a growing backlog never appears. Check whether flags_generated reconciles to the detection runs. |
| **Operated-but-ineffective** | Metrics are recovery-weighted only, so prevention - an edit that stops the claim, an education contact that changes behaviour - scores zero and gets defunded in favour of pay-and-chase. This is the failure mode with the largest long-run cost. |
| **Evidence-missing** | Flags are not linked to the test or model version that produced them, so no per-test precision can be computed at all and model monitoring is impossible. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 11. Cross-programme duplication - the same provider-period audited more than once

| | |
|---|---|
| **Paired control** | **PI-15** - Cross-contractor audit coordination & de-duplication |
| **Control type** | Preventive |
| **Module** | Module 02 - Program Integrity |
| **Owner** | PI coordinator |
| **Frequency** | Continuous |
| **System touchpoint** | Audit-tracking register |
| **Evidence artifact** | Coordination/de-dup log |
| **Control failure mode** | Duplicate/conflicting reviews; provider burden |
| **Regulatory anchor** | 42 CFR 455 Subpart F (Medicaid RAC) [binding]; GAO-23-106025 (28 Jun 2023) Rec 4 to CMS, open (managed-care RAC cost study); Recs 1 to 3 closed [case evidence] |
| **Secondary controls** | PI-13 - Contractor ADR-response tracking (UPIC/RAC/CERT/MAC); PI-19 - PI performance reporting (recovery ROI, prevention metrics) |

**1 · Data hypothesis.** Outlier lists are generated independently by several programmes - internal SIU, a recovery
contractor, a state unit, a plan's own analytics - and they all rank on roughly the same
signal, so they converge on the same providers. Without de-duplication the same provider-period
is requested, reviewed and sometimes recovered against more than once. This has three costs,
and the third is the one that gets missed: wasted review capacity, an unreasonable burden on
the provider, and a real risk of double recovery on the same dollars, which is a finding
against the payer rather than the provider.

**2 · What to check.** Across all audit and review programmes, find provider-period combinations touched by more than
one programme, and among those find any where recovered dollars overlap on the same claims.
Report burden separately from double recovery - the first is an efficiency and fairness
question, the second is a correction the payer owes.

**3 · Reference SQL.**

```sql
-- TEST 11 - Cross-programme audit duplication and potential double recovery
-- Runs on your audit-tracking and recovery systems.
-- Params: :period_start, :period_end
-- Portability: DATE_TRUNC('quarter', d) and STRING_AGG(DISTINCT ...) vary by engine; use year and quarter columns and LISTAGG or GROUP_CONCAT.

WITH touches AS (
  SELECT
      a.provider_npi,
      DATE_TRUNC('quarter', a.service_period_start)     AS service_quarter,
      a.programme,
      a.audit_id,
      a.dollars_identified,
      a.dollars_recovered
  FROM audit_activity a
  WHERE a.activity_date BETWEEN :period_start AND :period_end
),
dupes AS (
  SELECT provider_npi, service_quarter,
         COUNT(DISTINCT programme)  AS programmes,
         COUNT(DISTINCT audit_id)   AS audits,
         SUM(dollars_identified)    AS total_identified,
         SUM(dollars_recovered)     AS total_recovered
  FROM touches
  GROUP BY provider_npi, service_quarter
  HAVING COUNT(DISTINCT programme) > 1
)
SELECT
    d.provider_npi,
    d.service_quarter,
    d.programmes,
    d.audits,
    d.total_identified,
    d.total_recovered,
    STRING_AGG(DISTINCT t.programme, ', ')  AS programme_list
FROM dupes d
JOIN touches t
  ON t.provider_npi    = d.provider_npi
 AND t.service_quarter = d.service_quarter
GROUP BY d.provider_npi, d.service_quarter, d.programmes, d.audits,
         d.total_identified, d.total_recovered
ORDER BY d.audits DESC, d.total_recovered DESC;

-- The serious pass: same CLAIM recovered by two programmes. Run this one first if your
-- recovery records carry claim identifiers; a hit here is a payer-side correction.
--   SELECT r.claim_id, COUNT(DISTINCT r.programme) AS programmes,
--          SUM(r.amount_recovered) AS recovered_total
--   FROM recovery_detail r
--   WHERE r.recovery_date BETWEEN :period_start AND :period_end
--   GROUP BY r.claim_id
--   HAVING COUNT(DISTINCT r.programme) > 1
--   ORDER BY recovered_total DESC;
```

**4 · Threshold guidance.** Duplication is not a threshold test: any same-claim double recovery is a finding at a count of
one and should be worked immediately. Provider-period overlap between programmes is expected at
some level and only becomes a coordination finding when it is routine, so measure your own
overlap rate first and set the trigger relative to that baseline rather than importing a
number. A useful secondary measure is requests per provider per year across all programmes  - 
the burden number, which nobody computes and every provider association can. Tune to your
population; a starting point, not a standard.

**5 · Paired control + failure mode.** **PI-15 - cross-contractor audit coordination and de-duplication.** The test is the direct measurement of what PI-15 exists to prevent. Note the asymmetry in how the findings resolve: overlap is a coordination fix, double recovery is a repayment the payer owes, and conflating them in a single report lets the second hide inside the first.

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No coordination mechanism exists between programmes; each maintains its own worklist and there is no shared view of who is already under review. |
| **Designed-not-operating** | A coordination log or clearance step is defined but is not being populated, so clearance checks pass trivially because the log is empty. |
| **Operated-but-bypassed** | Coordination covers contractor programmes but not internal SIU work or delegated-entity audits, so the overlap simply moves to the uncovered channel. |
| **Operated-but-ineffective** | De-duplication matches on provider and date range but not on claim, so two programmes can recover different amounts against the same claims without triggering anything. |
| **Evidence-missing** | Recovery records lack claim-level detail, so double recovery cannot be detected or disproved - which is also the state in which a provider's assertion of it cannot be rebutted. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 12. Did the intervention work? Pre/post re-measurement of the flagged behaviour

| | |
|---|---|
| **Paired control** | **PI-11** - Corrective-action effectiveness re-measurement (loop closure) |
| **Control type** | Detective |
| **Module** | Module 02 - Program Integrity |
| **Owner** | PI / compliance |
| **Frequency** | Quarterly |
| **System touchpoint** | CAPA tracker + analytics |
| **Evidence artifact** | Re-measurement / trend report |
| **Control failure mode** | Ineffective fixes left in place |
| **Regulatory anchor** | GAO-19-277 (27 Mar 2019) Recs 1 and 2 to CMS, open [case evidence] |
| **Secondary controls** | PI-10 - Root-cause corrective-action workflow (beyond recoupment); MD-13 - Improper-payment corrective action plans addressing root causes, … |

**1 · Data hypothesis.** Corrective action is treated as complete when it is delivered rather than when it works. A
provider is educated, an edit is installed, a recovery is taken - and no one returns to measure
whether the underlying rate changed. Where the effectiveness re-measurement loop is absent, the
observable signature is a set of providers whose post-intervention metric is statistically
indistinguishable from their pre-intervention metric, sitting in a case system where every one
of those cases is marked closed and effective. The mirror signature matters too: interventions
that worked and were never credited, which is how effective prevention work loses its funding
to recovery work.

**2 · What to check.** For each closed corrective action, recompute the provider's flagged metric over a matched
window before and after the intervention date, and compare the change against the change in a
peer group over the same calendar window - so a market-wide shift is not credited to the
intervention. Classify each into improved, unchanged or worsened, and report the distribution
by intervention type.

**3 · Reference SQL.**

```sql
-- TEST 12 - Pre/post effectiveness re-measurement with a peer-difference correction
-- Runs on your case system joined to the metric that produced the flag.
-- Params: :window_days (180), :improve_cut (0.20)
-- Portability: date +/- integer days is not universal; use DATEADD(day, n, d) or d + INTERVAL n DAY. NULLS LAST: sort on (col IS NULL) first.

WITH actions AS (
  SELECT ca.action_id, ca.provider_npi, ca.action_type,      -- EDUCATION / EDIT / RECOVERY / CAP
         ca.intervention_date, ca.metric_id, ca.closed_date, ca.closure_assessment
  FROM corrective_actions ca
  WHERE ca.closed_date IS NOT NULL
),
metric AS (
  -- provider_metric_history is whatever table stores the recurring value of a detection
  -- metric per provider per period. If you do not have one, build it: without a metric
  -- history this control cannot be tested at all, which is itself the finding.
  SELECT m.provider_npi, m.metric_id, m.period_end, m.metric_value, m.peer_mean
  FROM provider_metric_history m
),
prepost AS (
  SELECT
      a.action_id, a.provider_npi, a.action_type, a.intervention_date, a.closure_assessment,
      AVG(CASE WHEN m.period_end <  a.intervention_date
                AND m.period_end >= a.intervention_date - :window_days
               THEN m.metric_value END)                        AS pre_value,
      AVG(CASE WHEN m.period_end >  a.intervention_date
                AND m.period_end <= a.intervention_date + :window_days
               THEN m.metric_value END)                        AS post_value,
      AVG(CASE WHEN m.period_end <  a.intervention_date
                AND m.period_end >= a.intervention_date - :window_days
               THEN m.peer_mean END)                           AS pre_peer,
      AVG(CASE WHEN m.period_end >  a.intervention_date
                AND m.period_end <= a.intervention_date + :window_days
               THEN m.peer_mean END)                           AS post_peer
  FROM actions a
  JOIN metric m ON m.provider_npi = a.provider_npi AND m.metric_id = a.metric_id
  GROUP BY a.action_id, a.provider_npi, a.action_type, a.intervention_date, a.closure_assessment
)
SELECT
    action_id, provider_npi, action_type, intervention_date, closure_assessment,
    ROUND(pre_value, 3)   AS pre_value,
    ROUND(post_value, 3)  AS post_value,
    -- difference-in-differences: the provider's change net of the peer group's change
    ROUND((post_value - pre_value) - (post_peer - pre_peer), 3) AS peer_adjusted_change,
    CASE
      WHEN post_value IS NULL OR pre_value IS NULL THEN 'NOT_MEASURABLE'
      WHEN ((post_value - pre_value) - (post_peer - pre_peer))
           / NULLIF(pre_value, 0) <= -:improve_cut               THEN 'IMPROVED'
      WHEN ((post_value - pre_value) - (post_peer - pre_peer))
           / NULLIF(pre_value, 0) >=  :improve_cut               THEN 'WORSENED'
      ELSE 'UNCHANGED'
    END                                                          AS effectiveness,
    CASE WHEN closure_assessment = 'EFFECTIVE'
          AND (post_value IS NULL OR pre_value IS NULL
               OR ((post_value - pre_value) - (post_peer - pre_peer))
                  / NULLIF(pre_value, 0) > -:improve_cut)
         THEN 'CLOSURE_NOT_SUPPORTED' END                        AS closure_check
FROM prepost
ORDER BY peer_adjusted_change DESC NULLS LAST;

-- CLOSURE_NOT_SUPPORTED is the column to read first: cases the programme certified as
-- effective where the data does not show improvement. It is a finding about the closure
-- process, not about the provider.
```

**4 · Threshold guidance.** :window_days = 180 balances having enough post-intervention data against waiting so long that
other changes intervene; shorten it for edits, which act immediately, and lengthen it for
education, which does not. :improve_cut = 0.20 (a 20% peer-adjusted movement) is a starting
point chosen to be larger than ordinary period-to-period noise - measure your metric's own
volatility and set the cut above it, or the test will report improvement and deterioration at
random. The peer-difference correction is not optional: without it, a market-wide coding change
is credited to whatever intervention happened to precede it. Tune to your population; a
starting point, not a standard.

**5 · Paired control + failure mode.** **PI-11 - corrective-action effectiveness re-measurement (loop closure).** This is the control the whole family exists to serve: detection without effectiveness measurement is a treadmill. PI-10 is the root-cause workflow that produces the action; MD-13 is the Medicaid improper-payment CAP-to-closure requirement. Note that NOT_MEASURABLE rows are as informative as the rest - they identify actions taken against a metric that is not tracked over time, so effectiveness was never knowable.

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No re-measurement occurs; cases close on delivery of the action. Look for closure assessments that are uniformly 'effective' with no supporting metric. |
| **Designed-not-operating** | Re-measurement is defined in the PI plan at a stated interval but no re-measurement records exist for the period. |
| **Operated-but-bypassed** | Re-measurement is performed for edits and system changes, where it is easy, and skipped for provider-level education, where most of the case volume is. |
| **Operated-but-ineffective** | Re-measurement compares raw before and after with no peer adjustment, so it credits or blames the intervention for market-wide movement. The peer-adjusted column exists to expose exactly this. |
| **Evidence-missing** | No provider metric history is retained, so pre-intervention values cannot be reconstructed and the loop cannot be closed even retrospectively. Retaining the metric history is the cheapest remediation in this family. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 13. Education without effect - repeat outliers across consecutive detection cycles

| | |
|---|---|
| **Paired control** | **PI-12** - Provider education & error-trend feedback |
| **Control type** | Preventive |
| **Module** | Module 02 - Program Integrity |
| **Owner** | Provider education |
| **Frequency** | Quarterly |
| **System touchpoint** | Education/outreach log |
| **Evidence artifact** | Education notice + acknowledgment |
| **Control failure mode** | Recurring billing errors |
| **Regulatory anchor** | CMS PIM Pub 100-08 Ch.3 (Verifying Errors & Corrective Actions) [guidance]; CMS CERT Program [case evidence] |
| **Secondary controls** | PI-11 - Corrective-action effectiveness re-measurement (loop closure); CB-19 - Coding audit program (statistically valid sampling) + error-rate … |

**1 · Data hypothesis.** Provider education is the cheapest corrective action and therefore the default one, which makes
it the action most likely to be applied where it cannot work. Education addresses error; it
does nothing about deliberate conduct and nothing about a structural incentive. Where the
education-and-feedback control operates without an escalation rule, the signature is a small
population of providers who appear on the outlier list every cycle, receive an education
contact every cycle, and never move - and whose repeat appearances are counted as evidence the
education programme is busy. The number of prior contacts is the variable that should drive
escalation and usually is not recorded at all.

**2 · What to check.** Join the outlier list across consecutive detection cycles and count consecutive appearances per
provider. For each repeat provider, count prior education contacts and any escalation beyond
education. Flag providers above an appearance count who have received only education, and
report the dollars flowing through them while the cycle repeats.

**3 · Reference SQL.**

```sql
-- TEST 13 - Repeat outliers who have received only education
-- Runs on the flag history joined to the contact/action log.
-- Params: :cycles_lookback (4), :repeat_cut (3), :education_cut (2)
-- Portability: LIMIT inside a subquery is not universal; use FETCH FIRST n ROWS ONLY, TOP n, or ROW_NUMBER() OVER (ORDER BY cycle_end DESC) <= n.

WITH appearances AS (
  SELECT
      fl.provider_npi,
      fl.detection_cycle,                                -- e.g. 2026Q1, 2026Q2
      MAX(fl.dollars_at_risk)                            AS cycle_dollars
  FROM pi_flags fl
  WHERE fl.detection_cycle IN (SELECT cycle_id FROM detection_cycles
                               ORDER BY cycle_end DESC LIMIT :cycles_lookback)
  GROUP BY fl.provider_npi, fl.detection_cycle
),
repeats AS (
  SELECT provider_npi,
         COUNT(DISTINCT detection_cycle) AS cycles_flagged,
         SUM(cycle_dollars)              AS dollars_across_cycles
  FROM appearances
  GROUP BY provider_npi
  HAVING COUNT(DISTINCT detection_cycle) >= :repeat_cut
),
actions AS (
  SELECT ca.provider_npi,
         SUM(CASE WHEN ca.action_type = 'EDUCATION' THEN 1 ELSE 0 END) AS education_contacts,
         SUM(CASE WHEN ca.action_type IN ('AUDIT','RECOVERY','EDIT','REFERRAL','SUSPENSION')
                  THEN 1 ELSE 0 END)                                   AS escalated_actions,
         MAX(ca.intervention_date)                                     AS last_action_date
  FROM corrective_actions ca
  GROUP BY ca.provider_npi
)
SELECT
    r.provider_npi,
    r.cycles_flagged,
    r.dollars_across_cycles,
    COALESCE(a.education_contacts, 0)  AS education_contacts,
    COALESCE(a.escalated_actions, 0)   AS escalated_actions,
    a.last_action_date,
    CASE
      WHEN COALESCE(a.escalated_actions, 0) > 0                  THEN 'ESCALATED'
      WHEN COALESCE(a.education_contacts, 0) >= :education_cut   THEN 'EDUCATION_ONLY_REPEAT'
      WHEN COALESCE(a.education_contacts, 0) = 0                 THEN 'NO_ACTION_TAKEN'
      ELSE 'EDUCATION_BELOW_TRIGGER'   -- some education, fewer contacts than the trigger, nothing else
    END                                AS action_profile
FROM repeats r
LEFT JOIN actions a ON a.provider_npi = r.provider_npi
ORDER BY r.dollars_across_cycles DESC;

-- NO_ACTION_TAKEN is usually the larger population and the quieter problem: providers
-- flagged in three or four cycles with no recorded contact of any kind.
-- Report the two profiles separately; they have different remedies.
-- cycles_flagged counts distinct flagged cycles inside the lookback. It does not require
-- them to be consecutive; add a gap check if consecutive appearance matters to you.
```

**4 · Threshold guidance.** :repeat_cut = 3 consecutive-or-near-consecutive cycles is a starting point and depends entirely
on cycle length - three quarterly cycles is nine months of unchanged behaviour, three monthly
cycles is not much at all. The query counts flagged cycles inside the lookback and does not
require them to be consecutive. :education_cut = 2 education contacts with no escalation is
offered as the trigger for an escalation review, not as a rule that education has failed; some
legitimate practice patterns will recur permanently and the right outcome for those is a
documented exclusion rather than a third contact. Tune to your population; a starting point,
not a standard.

**5 · Paired control + failure mode.** **PI-12 - provider education and error-trend feedback.** The control is sound and the test is about its boundary: education is the correct response to error and the wrong response to everything else, so a programme with no escalation rule converts every case into an error case by default. Pair with PI-11 (was the education effective) and CB-19 (coding audit) as the usual next step.

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No education or feedback programme exists; providers learn about their outlier status only through an audit, which is both more expensive and more adversarial than it needs to be. |
| **Designed-not-operating** | An education programme is described in the compliance plan but no contacts were logged in the period. |
| **Operated-but-bypassed** | Education is delivered to small providers and skipped for the large systems that generate the most dollars, usually because the relationship is managed elsewhere in the organisation. |
| **Operated-but-ineffective** | Education is delivered repeatedly with no escalation criterion, so the same providers cycle indefinitely. EDUCATION_ONLY_REPEAT is the direct measurement of this failure. |
| **Evidence-missing** | Contacts are not logged against the provider record, so prior-contact count is unknown and escalation cannot be triggered by it even if a rule exists. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 14. Identified but not recovered - overpayment aging against the look-back limit

| | |
|---|---|
| **Paired control** | **PI-09** - Overpayment recovery tracking vs reopening / RAC look-back |
| **Control type** | Detective |
| **Module** | Module 02 - Program Integrity |
| **Owner** | PI recovery analyst |
| **Frequency** | Continuous |
| **System touchpoint** | Recovery/AR system |
| **Evidence artifact** | Recovery aging report |
| **Control failure mode** | Missed/time-barred recoveries |
| **Regulatory anchor** | CMS PIM Pub 100-08 Ch.8 (Administrative Actions / Sampling) [guidance]; 42 CFR 405.980 (reopening) / SSA 1893(h) (RAC) [binding] |
| **Secondary controls** | PI-07 - 60-day self-identified overpayment report-&-return workflow; MC-19 - 60-day overpayment identification, refund & self-disclosure pathway |

**1 · Data hypothesis.** Dollars identified is the number every programme reports; dollars recovered is the number that
changes the ledger, and the gap between them is where the recovery control fails silently. Two
mechanisms produce the gap. Cases age past the reopening or look-back limit while sitting in a
queue, at which point the money is gone as a matter of law rather than as a matter of
collectability. And identified overpayments are recorded but never converted into a demand, so
nothing ever starts. The second is harder to see because the case looks open and healthy right
up until it expires.

**2 · What to check.** For each identified overpayment, compute age from identification, whether a demand was issued,
days remaining against the applicable look-back or reopening limit, and amounts outstanding.
Bucket by time-to-expiry and report dollars in each bucket, so the report says how much money
will be lost in the next quarter if nothing changes.

**3 · Reference SQL.**

```sql
-- TEST 14 - Overpayment aging against the look-back limit
-- Runs on your overpayment / recovery tracking system.
-- Params: :lookback_days (1460 for a four-year reopening window - use YOUR applicable limit)
-- Portability: DATE_DIFF with a trailing DAY unit is BigQuery syntax (elsewhere DATEDIFF(day, earlier, later)); GROUP BY 2 is an ordinal, repeat the CASE where ordinals are not accepted.

WITH op AS (
  SELECT
      o.overpayment_id,
      o.provider_npi,
      o.source,                                        -- SELF_REPORTED / AUDIT / ANALYTICS / CONTRACTOR
      o.service_period_end,
      o.identified_date,
      o.demand_issued_date,
      o.amount_identified,
      COALESCE(o.amount_recovered, 0)                  AS amount_recovered,
      o.status,
      DATE_DIFF(CURRENT_DATE, o.identified_date, DAY)  AS days_since_identified,
      :lookback_days - DATE_DIFF(CURRENT_DATE, o.service_period_end, DAY)
                                                       AS days_to_lookback_expiry
  FROM overpayments o
  -- a NULL status is unknown, not recovered: keep the row (a bare <> would drop it)
  WHERE COALESCE(o.status, 'UNKNOWN') <> 'FULLY_RECOVERED'
)
SELECT
    source,
    CASE
      WHEN days_to_lookback_expiry <  0   THEN 'EXPIRED'
      WHEN days_to_lookback_expiry <  90  THEN 'EXPIRES_WITHIN_90D'
      WHEN days_to_lookback_expiry < 365  THEN 'EXPIRES_WITHIN_1Y'
      ELSE 'IN_WINDOW'
    END                                                        AS expiry_bucket,
    COUNT(*)                                                   AS cases,
    SUM(CASE WHEN status IS NULL THEN 1 ELSE 0 END)            AS cases_status_unknown,
    SUM(amount_identified - amount_recovered)                  AS outstanding,
    SUM(CASE WHEN demand_issued_date IS NULL THEN 1 ELSE 0 END) AS no_demand_issued,
    SUM(CASE WHEN demand_issued_date IS NULL
             THEN amount_identified - amount_recovered END)     AS outstanding_no_demand,
    ROUND(AVG(days_since_identified), 0)                        AS mean_age_days
FROM op
GROUP BY source, 2
ORDER BY outstanding DESC;

-- The line to act on is EXPIRES_WITHIN_90D with no demand issued. Everything in it is
-- recoverable today and unrecoverable next quarter, and nothing else in this pack has
-- that property.
--
-- Separately: the 60-day clock on self-identified overpayments (PI-07 / MC-19) runs from
-- identification, not from the look-back limit, and is much shorter. Report source =
-- 'SELF_REPORTED' against a 60-day age, not against this bucket scheme.
```

**4 · Threshold guidance.** :lookback_days must be replaced with your own applicable limit - reopening rules, state
recovery statutes and contract terms all differ, and the four-year value used here is a
placeholder chosen only to make the query runnable. The 90-day bucket is the operationally
useful one because it matches a quarter of management attention. No threshold in this test is
statistical; every one is a legal or contractual deadline, so confirm each against its source
rather than tuning it. Where a demand tolls or extends the limit in your programme, model that
explicitly rather than using identified date as a proxy.

**5 · Paired control + failure mode.** **PI-09 - overpayment recovery tracking versus reopening / RAC look-back.** The test is a direct instrument for the control. PI-07 and MC-19 (60-day identify-report-return obligations) are the much tighter clock on self-identified overpayments and should be measured separately; mixing the two populations in one aging report is the most common way the 60-day exposure gets hidden inside a comfortable-looking four-year window.

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No aging of identified overpayments against any limit; recovery is worked in whatever order cases arrive and expiry is discovered after the fact. |
| **Designed-not-operating** | An aging report is specified but not produced, or is produced without the look-back comparison, which is the only part that matters. |
| **Operated-but-bypassed** | Aging covers contractor-identified overpayments, which come with their own tracking, and omits internally identified and self-reported ones. |
| **Operated-but-ineffective** | Aging runs from identification date rather than from the service period, so a case identified late looks young while its look-back window is already closing. Compare the two age columns; the divergence is the failure. |
| **Evidence-missing** | Demand issuance is not recorded against the overpayment, so it is impossible to show whether recovery was ever pursued - and equally impossible to defend the programme against a claim that it was not. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 15. Payment velocity during an open investigation - exposure accruing while a case runs

| | |
|---|---|
| **Paired control** | **PI-16** - Payment suspension on credible allegation of fraud |
| **Control type** | Preventive |
| **Module** | Module 02 - Program Integrity |
| **Owner** | SIU / payment integrity |
| **Frequency** | Per case |
| **System touchpoint** | Payment-suspension system |
| **Evidence artifact** | Suspension determination memo |
| **Control failure mode** | Continued payment during active fraud |
| **Regulatory anchor** | 42 CFR 455.23 (Medicaid payment suspension) [binding]; 42 CFR 405.371 (Medicare payment suspension) [binding] |
| **Secondary controls** | MD-18 - Credible-allegation-of-fraud payment suspension + MFCU referral; PI-14 - Suspected-fraud referral to OIG/MFCU/law enforcement |

**1 · Data hypothesis.** Between the moment a credible allegation exists and the moment anything is done about it,
claims continue to pay at whatever rate they were paying. Where the payment-suspension control
is absent or is treated as a last resort rather than a considered decision, the observable
signature is measurable and rarely measured: dollars paid to providers with open high-severity
cases, accruing over the life of the case. The right output is emphatically not a suspension
recommendation - suspension is a serious action with due-process requirements, real
consequences for beneficiary access, and a legal standard this query cannot evaluate. The
output is the exposure number that should be in front of whoever makes that decision.

**2 · What to check.** For each open case at or above a defined severity, compute dollars paid to the subject provider
between the case-open date and today, the daily payment rate, and whether a suspension,
prepayment review or other mitigation was considered and recorded. Report accrued exposure and
the decision record, not a recommendation.

**3 · Reference SQL.**

```sql
-- TEST 15 - Exposure accruing during open investigations
-- Runs on the case system joined to paid-claims totals. NOT a suspension recommendation.
-- Params: :severity_min ('HIGH'), :period_end
-- Portability: DATE_DIFF with a trailing DAY unit is BigQuery syntax (elsewhere DATEDIFF(day, earlier, later)); where there is no boolean type compare the flags to 1 and 0.

WITH ranked AS (
  SELECT c.case_id, c.provider_npi, c.severity, c.opened_date,
         c.suspension_considered_flag, c.suspension_decision,
         c.prepay_review_flag, c.referral_date,
         -- severity is text: compare ranks, never the strings (alphabetically CRITICAL < HIGH < LOW < MEDIUM)
         -- adapt both lists to your severity scheme; a value on neither list ranks NULL
         CASE c.severity    WHEN 'LOW' THEN 1 WHEN 'MEDIUM' THEN 2 WHEN 'HIGH' THEN 3 WHEN 'CRITICAL' THEN 4 END AS severity_rank,
         CASE :severity_min WHEN 'LOW' THEN 1 WHEN 'MEDIUM' THEN 2 WHEN 'HIGH' THEN 3 WHEN 'CRITICAL' THEN 4 END AS min_rank
  FROM pi_cases c
  WHERE c.status = 'OPEN'
),
open_cases AS (
  SELECT case_id, provider_npi, severity, opened_date,
         suspension_considered_flag, suspension_decision, prepay_review_flag, referral_date,
         CASE WHEN min_rank IS NULL      THEN 'SEVERITY_MIN_NOT_RECOGNISED'
              WHEN severity_rank IS NULL THEN 'UNKNOWN_SEVERITY'
              ELSE 'OK' END                            AS severity_check
  FROM ranked
  WHERE severity_rank >= min_rank
     OR severity_rank IS NULL     -- missing or unlisted severity: kept and labelled, never dropped
     OR min_rank IS NULL          -- unlisted :severity_min: nothing is filtered and every row says so
),
paid_since AS (
  SELECT oc.case_id,
         SUM(cl.paid_amount)                                     AS paid_since_open,
         COUNT(*)                                                AS lines_since_open,
         MIN(cl.service_date)                                    AS first_service,
         MAX(cl.service_date)                                    AS last_service
  FROM open_cases oc
  JOIN claim_lines cl
    ON cl.billing_npi = oc.provider_npi
   AND cl.paid_date BETWEEN oc.opened_date AND :period_end
   AND cl.adjudication_status = 'PAID'
  GROUP BY oc.case_id
)
SELECT
    oc.case_id,
    oc.provider_npi,
    oc.severity,
    oc.severity_check,
    oc.opened_date,
    DATE_DIFF(:period_end, oc.opened_date, DAY)                          AS case_age_days,
    COALESCE(p.paid_since_open, 0)                                       AS paid_since_open,
    ROUND(COALESCE(p.paid_since_open, 0)
          / NULLIF(DATE_DIFF(:period_end, oc.opened_date, DAY), 0), 2)   AS daily_accrual,
    p.lines_since_open,
    oc.suspension_considered_flag,
    oc.suspension_decision,
    oc.prepay_review_flag,
    oc.referral_date,
    CASE WHEN COALESCE(oc.suspension_considered_flag, FALSE) = FALSE
          AND COALESCE(oc.prepay_review_flag, FALSE) = FALSE
         THEN 'NO_MITIGATION_RECORDED' END                                AS mitigation_gap
FROM open_cases oc
LEFT JOIN paid_since p ON p.case_id = oc.case_id
ORDER BY paid_since_open DESC NULLS LAST;

-- Read mitigation_gap as a DOCUMENTATION finding. A decision not to suspend is a
-- legitimate and often correct decision; a case where nobody recorded making the
-- decision either way is the finding. The regulation contemplates good-cause exceptions
-- and they must be documented to exist.
-- Read severity_check before anything else: UNKNOWN_SEVERITY rows carry a severity this
-- query does not rank (fix the record or the list), and SEVERITY_MIN_NOT_RECOGNISED means
-- the parameter itself is not on the list, so the severity filter did not run at all.
```

**4 · Threshold guidance.** There is no dollar threshold here on purpose. Any figure would read as an implicit trigger for
suspension, and suspension is not a threshold decision - it turns on the credibility of the
allegation and on documented good-cause considerations, neither of which is in claims data.
What can be set locally is a review trigger: an accrued exposure or case age at which the
mitigation decision must be revisited and recorded. Also worth measuring is the distribution of
case age at severity, because the real problem is usually case cycle time rather than the
suspension decision itself. Tune to your population; a starting point, not a standard.

**5 · Paired control + failure mode.** **PI-16 - payment suspension on credible allegation of fraud.** Note the direction of the pairing carefully: the test does not measure whether suspension happened, it measures whether the decision was made and recorded, and what it cost while it was pending. MD-18 is the Medicaid credible-allegation suspension and MFCU referral control; PI-14 is the law-enforcement referral pathway, and the referral date column is included because a pending referral is one of the recognised good-cause grounds for not suspending.

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No suspension authority is operationalised - the control exists in regulation but there is no internal process to invoke it, so the option is never on the table. |
| **Designed-not-operating** | A suspension process exists but has not been invoked or formally declined in any case in the period; check whether the consideration step appears anywhere in the case workflow. |
| **Operated-but-bypassed** | Suspension is considered only on referred cases and never on internally developed ones, or is routinely deferred pending a referral that is not pursued. |
| **Operated-but-ineffective** | Suspension decisions are made so late in the case lifecycle that most of the exposure has already accrued. The daily-accrual column times the median case age is the size of this failure. |
| **Evidence-missing** | Neither the decision nor the good-cause rationale is recorded, so the payer cannot show it considered suspension - the position that is hardest to defend afterwards, whichever way the decision went. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 16. Detection coverage map - which spend has no test looking at it

| | |
|---|---|
| **Paired control** | **MD-19** - CMIP-aligned PI unit with analytics & UPIC coordination |
| **Control type** | Preventive |
| **Module** | Module 07 - Medicaid & CHIP Integrity |
| **Owner** | Medicaid PI director |
| **Frequency** | Annual |
| **System touchpoint** | Governance & analytics platform |
| **Evidence artifact** | PI plan + UPIC coordination MOU |
| **Control failure mode** | Fragmented PI without analytics capacity |
| **Regulatory anchor** | SSA sec.1936 [binding]; CMS Comprehensive Medicaid Integrity Plan (CMIP) [guidance] |
| **Secondary controls** | PI-01 - Routine claims-aberrancy analytics (peer-outlier / statistical); PI-18 - PI/SIU governance: roles, SoD & independence from payment |

**1 · Data hypothesis.** Detection programmes accrete. Tests are written in response to whatever went wrong last, so
coverage follows history rather than exposure, and the categories that have never produced a
scandal are the categories nobody watches. Where the programme has no coverage map, the
signature is straightforward and slightly uncomfortable to look at: rank spend by service
category, mark which categories had a detection test run against them in the period, and read
the unmarked rows. That list is the programme's blind spot, and it is usually larger and more
expensive than the people running the programme expect - not because anyone decided not to
look, but because nobody ever laid the two lists side by side.

**2 · What to check.** Aggregate paid dollars by service category and provider type for the period. Join to the
inventory of detection tests actually executed, by the category each test covers. Report every
category with material spend and no executed test, ranked by dollars, and compute the share of
total spend under active detection coverage.

**3 · Reference SQL.**

```sql
-- TEST 16 - Coverage map: spend by category against detection tests actually run
-- Runs on claims aggregates joined to your detection-test inventory/run log.
-- Params: :period_start, :period_end, :material_pct (0.005)
-- Portability: STRING_AGG(DISTINCT ...) is LISTAGG or GROUP_CONCAT on other engines.

WITH spend AS (
  SELECT
      cl.service_category,
      cl.provider_type,
      SUM(cl.paid_amount)                                       AS paid_total,
      COUNT(DISTINCT cl.billing_npi)                            AS providers,
      SUM(cl.paid_amount) / SUM(SUM(cl.paid_amount)) OVER ()    AS share_of_spend
  FROM claim_lines cl
  WHERE cl.service_date BETWEEN :period_start AND :period_end
    AND cl.adjudication_status = 'PAID'
  GROUP BY cl.service_category, cl.provider_type
),
covered AS (
  SELECT tc.service_category, tc.provider_type,
         COUNT(DISTINCT tr.run_id)  AS runs_in_period,
         MAX(tr.run_date)           AS last_run_date,
         -- only tests with a run in the period count as covering; the rest are listed apart
         STRING_AGG(DISTINCT CASE WHEN tr.run_id IS NOT NULL THEN tc.detection_test_id END, ', ')
                                    AS tests_run,
         STRING_AGG(DISTINCT CASE WHEN tr.run_id IS NULL THEN tc.detection_test_id END, ', ')
                                    AS tests_not_run
  FROM detection_test_coverage tc                     -- which categories each test covers
  LEFT JOIN detection_test_runs tr
         ON tr.detection_test_id = tc.detection_test_id
        AND tr.run_date BETWEEN :period_start AND :period_end
  GROUP BY tc.service_category, tc.provider_type
)
SELECT
    s.service_category,
    s.provider_type,
    s.paid_total,
    s.providers,
    ROUND(100.0 * s.share_of_spend, 2)                 AS pct_of_spend,
    COALESCE(c.runs_in_period, 0)                      AS detection_runs,
    c.last_run_date,
    COALESCE(c.tests_run, '(none)')                    AS tests_covering,
    COALESCE(c.tests_not_run, '(none)')                AS tests_defined_not_run,
    CASE
      WHEN c.service_category IS NULL              THEN 'NO_TEST_DEFINED'
      WHEN COALESCE(c.runs_in_period, 0) = 0       THEN 'TEST_DEFINED_NOT_RUN'
      ELSE 'COVERED'
    END                                                AS coverage_status
FROM spend s
LEFT JOIN covered c
       ON c.service_category = s.service_category
      AND c.provider_type    = s.provider_type
WHERE s.share_of_spend >= :material_pct
ORDER BY s.paid_total DESC;

-- Headline number worth putting on one slide: the share of paid dollars in categories
-- with coverage_status = 'COVERED'. Most programmes have never computed it, and the
-- first computation of it is usually the most useful output of this entire pack.
```

**4 · Threshold guidance.** :material_pct = 0.005 (half a percent of total spend) keeps the map readable while still
surfacing categories large enough to matter; lower it once the obvious gaps are closed. Do not
set a coverage target - a programme can reach high nominal coverage by defining one weak test
per category, and the honest reading of this map is qualitative: which uncovered categories
carry both material dollars and known scheme risk. Cross-reference the uncovered list against
the case base and against the other six families in this pack before deciding what to build
next. Tune to your population; a starting point, not a standard.

**5 · Paired control + failure mode.** **MD-19 - CMIP-aligned programme-integrity unit with analytics and UPIC coordination.** Coverage is the question a PI unit's design has to answer and the one least often asked of it: not whether the analytics are good, but whether they point at the money. PI-18 (PI/SIU governance, segregation of duties, independence from payment) is the secondary because a coverage map is only credible if the unit producing it is independent of the operations whose blind spots it reports.

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No detection-test inventory exists, so coverage cannot be stated at all and this query cannot be run. Building the inventory is the remediation and it is largely a documentation exercise. |
| **Designed-not-operating** | An inventory exists but no run log, so tests defined years ago cannot be distinguished from tests running today. TEST_DEFINED_NOT_RUN is the population to look at. |
| **Operated-but-bypassed** | Coverage is claimed at the programme level while whole delegated or carved-out benefits - behavioural health, dental, transport, pharmacy under a PBM - sit outside the analytics environment entirely and never appear in the spend denominator either. |
| **Operated-but-ineffective** | Coverage is broad and shallow: every category has a test, and most of those tests are threshold reports nobody works. Read this test with test 10 - coverage without yield is coverage on paper. |
| **Evidence-missing** | Test runs are not logged, so the programme cannot demonstrate what surveillance was in place during any past period, which is precisely the question asked after a scheme surfaces. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## Gap notes - controls this family needs that the library does not yet have

*Each note names a control objective these tests point to that the control library does not yet state. They are candidates for a future library release; the library itself is unchanged by this pack.*

### G1. No control owns peer-group construction, risk adjustment, or estimator choice - the assumptions every test in this family rests on.

PI-01 requires routine peer-outlier analytics and names the technique as 'peer/statistical',
and PI-02 requires a risk-scoring and prioritisation model. Neither states how a peer group is
to be defined, what minimum cell size makes a peer statistic meaningful, whether risk
adjustment is required, or how the estimator is chosen and validated. Those four decisions
determine whether every hit in tests 1, 3, 5, 6, 7 and 8 is real, and they are currently made
implicitly by whoever writes the query. Test 2 and test 4 exist because the library has no
control to point them at: test 2 is pointed at PE-02 (enrollment data quality), which owns the
taxonomy field but not its use as a comparison key, and test 4 is pointed at AG-01, which owns
model governance in general but does not reach statistical methodology. This is the clearest
structural gap the pack has surfaced so far, because it sits underneath six of the sixteen
tests in this family rather than beside them.

**Recommendation.** A peer-comparison methodology control for Module 02, stating four requirements: a documented
peer-group definition with a minimum cell size; risk adjustment using a named population risk
measure, or an explicit statement that the comparison is unadjusted; an estimator selected
against the actual peer distribution rather than assumed; and periodic revalidation of all
three. Cheap to write, cheap to assess, and it converts six analytics tests from ad hoc queries
into a testable control.

### G2. AG-01 is a single omnibus control carrying the entire analytics-governance load, and MEVIA-style operation needs it decomposed.

AG-01 bundles model inventory, intended-use statements, pre-deployment testing, a human-in-the-
loop gate before adverse action, and incident disclosure into one control - and it is the only
control in Module 08. Tests 4 and 10 both pair to it and are testing genuinely different
things: test 4 is pre-deployment methodology validation, test 10 is post-deployment performance
monitoring by model version. Neither drift monitoring, version control, feedback-loop cadence,
nor precision monitoring is separately stated, so an assessment can answer AG-01 affirmatively
on the strength of a model inventory alone while none of the ongoing monitoring exists.
Aligning Module 08 to MEVIA governance leads to the same conclusion from the strategy side, so
the two lines of reasoning corroborate each other.

**Recommendation.** Split AG-01 into a Module 08 control set mirroring MEVIA governance: model inventory and
intended use; pre-deployment validation including methodology; version control with flag-to-
version traceability; ongoing performance and drift monitoring with a stated cadence; human-in-
the-loop adjudication before any adverse action; and incident disclosure. Test 10's
`model_version` field on the flag record is the concrete artifact that makes the traceability
requirement assessable - worth naming in the control text.

### G3. MC-14 is the library's only beneficiary-level utilisation control and it is drug-specific.

Test 6 runs the family's arithmetic from the member side and has to pair to MC-14 (opioid
overutilisation and drug-management programme) because nothing broader exists. Service-side
overutilisation - a beneficiary receiving care from an implausible number of unrelated
providers, or utilisation far above risk-matched peers in non-pharmacy categories - has no
owning control at all. The gap has a sharp edge: the same pattern is the signature of a member
being used as a billing vehicle and of a member with fragmented care and no case management,
and without a control that states the care-coordination reading first, the analytics default to
an integrity reading. That is the beneficiary-harm mirror risk this pack has now flagged in
families 4, 6 and 7.

**Recommendation.** A beneficiary-level utilisation-review control, sited in Module 02 or Module 06, written with
the dual reading built into the control text: flags are routed to care management by default
and to integrity review only on a stated additional criterion. MC-14's existing beneficiary-
protection framing - lock-in requires notice, appeal rights and clinical review - is the
template, and MC-23 (two-sided under-service monitoring) shows the library already accepts
controls that look in both directions.

### G4. No control governs the intake of external vulnerability intelligence into local detection - the MEVIA loop itself.

CB-17 requires prepayment edit maintenance and tuning 'from emerging vulnerabilities' and is
the closest thing the library has, but it does not name a source, an intake route, or a time
limit for acting. MEVIA is explicitly a hub-and-spoke model - CMS detects a vulnerability,
hands it to a state, and tracks whether the state acted - and the spoke end of that loop is
unowned here. The same is true of the other external streams a payer receives: OIG work plan
items, UPIC referrals, NCCI updates, and fraud alerts from other payers. Test 16 measures
coverage but cannot measure responsiveness, because there is no record of what intelligence
arrived and what was done with it.

**Recommendation.** An intelligence-intake-and-action control for Module 02: a register of external vulnerability
intelligence received, with source, date, an assessment of local applicability, the action
taken or the documented reason for none, and a stated response interval. It makes the loop
assessable in both directions and gives test 16 a companion metric - time from intelligence
received to detection test deployed - which is the number MEVIA-style oversight will eventually
ask a payer for.

---

*Reference patterns. Public-source, personal-capacity, advisory and organizational only. Queries must be adapted to your own schema and tested on your own warehouse before use. Outputs are review candidates for human adjudication, never payment or denial dispositions. Confirm every regulatory anchor against its primary source.*