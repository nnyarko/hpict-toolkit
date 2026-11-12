# Analytics Test Pack - Prior authorisation and claims processing

*duplicate payment, PA-absent payment, denial quality, and the PERM data-processing surface*

*Generated 2026-11-11 from the HPI-CT control library (`data/controls.csv`, 173 controls). 16 detection tests, paired to 16 primary controls. Reference patterns, not programs.*

## Before you run anything

**These queries produce review candidates, never payment or denial decisions.** Every hit is a question for a human.

**The toolkit never receives your data.** These are patterns you run inside your own environment. No claim data, no PHI, no identifiers leave your systems.

**Every query is a pattern, not a program.** Field names are generic; yours will differ. Expect to rewrite the FROM and JOIN clauses and to remap service categories to your own taxonomy.

**Every threshold is a starting point, not a standard.** Where a threshold is arithmetic it is labelled as such. Everywhere else, measure your own distribution first.

Read `PREFACE_how_to_run_these.md` before the first run.

## Scope of this family

The general prior-authorisation controls (PA-03, PA-08 to PA-19) that family 6 deliberately
left out when it took the DMEPOS-specific subset, plus the claims-processing layer that sits
underneath them: duplicate detection, NCCI methodology coverage, pre-submission scrubbing, and
the PERM/MEQC data-processing error component. Every test in this family runs against the
payer's or plan's own adjudication, PA and appeals systems. Almost none of them is about a
provider.

This is the family where the subject under test is the payer. A duplicate payment is not
something a provider did to the plan; it is something the plan's duplicate key failed to see.
An untimely prior-authorisation decision, a denial issued with no clinician sign-off, an
internal criterion more restrictive than the national coverage determination it is supposed to
implement - each of these is a control failure whose evidence sits entirely inside the payer's
own systems, and each has a bright-line rule behind it rather than a statistical inference.
That makes the queries cheap and the findings hard to deflect. It also makes this the family
with the most beneficiary-harm exposure in the pack: eleven of the sixteen tests can surface
improper denial or delay, which is the direction of failure that costs a person their care
rather than costing the programme money. Both directions are reported here, and the family
refuses to treat only one of them as a finding.

## Tests at a glance

| # | Test | Primary control | Also touches |
|---|---|---|---|
| 1 | Duplicate payment - the four keys that a single duplicate rule cannot see | **CB-15** | CB-18, MD-12 |
| 2 | Medicaid NCCI - which of the six methodologies is actually firing | **CB-06** | CB-04, CB-07 |
| 3 | Clean-claim rate and the rework loop - the scrubber's real yield | **CB-18** | CB-04, PA-08 |
| 4 | PA decision timeliness - and what happens when the clock runs out | **PA-11** | PA-17, PA-12 |
| 5 | Denial notices that cannot be appealed - reason specificity and channel | **PA-12** | PA-11, PA-14 |
| 6 | Denial appropriateness - overturn rate as the outcome metric the sample should have found | **PA-14** | PA-15, PA-09 |
| 7 | Upheld denials that never reached an external reviewer | **PA-15** | PA-14, PA-19 |
| 8 | Denials with no human clinician behind them | **PA-19** | PA-12, PA-14 |
| 9 | Internal criteria more restrictive than the coverage rule they implement | **PA-09** | PA-14, PA-16 |
| 10 | Determinations made on incomplete utilisation data | **PA-10** | PA-03, PA-09 |
| 11 | Insufficient-documentation non-affirmation - whose package, whose checklist | **PA-03** | PA-08, PA-12 |
| 12 | High-denial categories with no documentation-eligibility checklist | **PA-08** | PA-03, CD-05 |
| 13 | EPSDT and children's services - PA as a barrier | **PA-16** | PA-14, PA-11 |
| 14 | PA metrics reporting - recompute the published numbers | **PA-17** | PA-11, PA-14 |
| 15 | PERM data-processing error surface - rebuild it before the cycle does | **MD-12** | CB-15, CB-06 |
| 16 | PTP edits that fire and are overridden - the bypass rate nobody reports | **CB-04** | CB-05, CB-17 |

> **Control-ID validation:** every control referenced above resolves to a real ID in the current control library. See the gap notes at the end for controls this family *needs* but the library does not yet have.

---

## 1. Duplicate payment - the four keys that a single duplicate rule cannot see

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
| **Secondary controls** | CB-18 - Pre-submission claim scrubber (provider side); MD-12 - PERM / MEQC eligibility & claims sampling with error-rate measurement |

**1 · Data hypothesis.** Duplicate detection is almost never absent; it is almost always keyed too narrowly. A rule
keyed on (claim number, line, provider, date, code, units) catches only resubmission of an
identical claim. It does not catch the same service billed under a different NPI in the same
group, under a different but clinically equivalent code, across a professional and a facility
claim, or across an FFS claim and an encounter already covered by capitation. Each of those is
a real duplicate with a different key, and each survives adjudication cleanly.

**2 · What to check.** Run four passes over the same paid population with four progressively looser keys, and report
each pass separately with its own count and exposure - never merged. The point is not a single
duplicate list; it is the shape of the gap between what the production rule catches and what
each looser key finds. That gap is the control finding, and it is stated in dollars.

**3 · Reference SQL.**

```sql
-- TEST 1 - Duplicate payment across four keys
-- Pattern only. Run on your own warehouse. Output = review candidates, not dispositions.
-- Params: :period_start, :period_end, :window_days (0 = same day only; 1 to 3 also reports exact-key repeats that many days apart)

WITH paid AS (
  SELECT cl.claim_line_id, cl.claim_id, cl.beneficiary_id, cl.service_date,
         cl.hcpcs, cl.modifier_1, cl.units, cl.paid_amount,
         cl.rendering_npi, cl.billing_npi, cl.group_tin,
         cl.claim_type,              -- 'PROFESSIONAL' | 'INSTITUTIONAL'
         cl.payer_program            -- 'FFS' | 'MANAGED_CARE' | waiver code
  FROM claim_lines cl
  WHERE cl.service_date BETWEEN :period_start AND :period_end
    AND cl.adjudication_status = 'PAID'
),

-- KEY 1 - exact duplicate: what your production rule should already catch.
-- Its purpose here is to establish the baseline. If this returns rows, the
-- production rule is not running, and no other pass means anything yet.
k1 AS (
  SELECT beneficiary_id, service_date, hcpcs, modifier_1, units,
         rendering_npi, billing_npi,
         COUNT(*) AS n, SUM(paid_amount) AS paid, MIN(claim_line_id) AS keep_line
  FROM paid
  GROUP BY beneficiary_id, service_date, hcpcs, modifier_1, units, rendering_npi, billing_npi
  HAVING COUNT(*) > 1
),

-- KEY 1, date drift - the exact key again, on service dates up to :window_days apart.
-- Reported as its own row so KEY1_EXACT stays arithmetic. Each row is a later line that
-- repeats an earlier one; with :window_days = 0 it returns nothing.
-- Date minus date gives days here (PostgreSQL, DuckDB, Oracle); elsewhere use DATEDIFF(day, a, b).
k1_drift AS (
  SELECT b.claim_line_id, b.paid_amount
  FROM paid b
  WHERE EXISTS (SELECT 1 FROM paid a
                WHERE a.beneficiary_id = b.beneficiary_id
                  AND a.hcpcs          = b.hcpcs
                  AND COALESCE(a.modifier_1, '') = COALESCE(b.modifier_1, '')
                  AND a.units          = b.units
                  AND a.rendering_npi  = b.rendering_npi
                  AND a.billing_npi    = b.billing_npi
                  AND a.service_date   < b.service_date
                  AND b.service_date - a.service_date <= :window_days)
),

-- KEY 2 - same service, different rendering NPI inside one group/TIN.
-- The classic locum, cross-cover and "billed by both partners" duplicate.
k2 AS (
  SELECT beneficiary_id, service_date, hcpcs, group_tin,
         COUNT(DISTINCT rendering_npi) AS distinct_renderers,
         COUNT(*) AS n, SUM(paid_amount) AS paid
  FROM paid
  WHERE group_tin IS NOT NULL
  GROUP BY beneficiary_id, service_date, hcpcs, group_tin
  HAVING COUNT(DISTINCT rendering_npi) > 1
),

-- KEY 3 - professional/institutional pairing for the same service.
-- Legitimate for split professional/technical component billing; the
-- discriminator is whether the modifiers (26 / TC) are present and consistent.
k3 AS (
  SELECT p.beneficiary_id, p.service_date, p.hcpcs,
         MAX(CASE WHEN p.claim_type = 'PROFESSIONAL'  THEN 1 ELSE 0 END) AS has_prof,
         MAX(CASE WHEN p.claim_type = 'INSTITUTIONAL' THEN 1 ELSE 0 END) AS has_inst,
         MAX(CASE WHEN p.modifier_1 IN ('26','TC')    THEN 1 ELSE 0 END) AS has_component_mod,
         SUM(p.paid_amount) AS paid, COUNT(*) AS n
  FROM paid p
  GROUP BY p.beneficiary_id, p.service_date, p.hcpcs
  HAVING MAX(CASE WHEN p.claim_type = 'PROFESSIONAL'  THEN 1 ELSE 0 END) = 1
     AND MAX(CASE WHEN p.claim_type = 'INSTITUTIONAL' THEN 1 ELSE 0 END) = 1
),

-- KEY 4 - FFS claim paid for a service already inside a capitated arrangement.
-- Usually the largest single bucket in a managed-care programme and the one
-- least likely to have any rule at all, because the two payment streams are
-- often held in different systems that are never joined.
k4 AS (
  SELECT p.beneficiary_id, p.service_date, p.hcpcs, p.paid_amount, p.claim_line_id
  FROM paid p
  JOIN capitation_enrollment ce
    ON  ce.beneficiary_id = p.beneficiary_id
    AND p.service_date BETWEEN ce.coverage_start AND COALESCE(ce.coverage_end, DATE '9999-12-31')
  WHERE p.payer_program = 'FFS'
    AND p.hcpcs IN (SELECT hcpcs FROM capitated_benefit_package WHERE package_id = ce.package_id)
)

-- Report each key separately. Do NOT union these into one duplicate list:
-- key 1 is an error, keys 2-4 are questions, and merging them destroys that.
SELECT 'KEY1_EXACT'          AS dup_key, COUNT(*) AS groups, SUM(paid) AS exposure FROM k1
UNION ALL
SELECT 'KEY1_DATE_DRIFT', COUNT(*), SUM(paid_amount) FROM k1_drift   -- counts later lines, not groups
UNION ALL
SELECT 'KEY2_CROSS_NPI_SAME_TIN', COUNT(*), SUM(paid) FROM k2
UNION ALL
SELECT 'KEY3_PROF_INST_PAIR',      COUNT(*), SUM(paid) FROM k3 WHERE has_component_mod = 0
UNION ALL
SELECT 'KEY4_FFS_INSIDE_CAPITATION', COUNT(*), SUM(paid_amount) FROM k4;

-- Then drill into whichever key carries material exposure. The control finding is
-- the SHAPE of the gap: "the production key catches KEY1 and nothing else, and
-- KEY4 carries $X" is a specification defect stated in dollars.
```

**4 · Threshold guidance.** No statistical threshold - KEY1 is arithmetic and any row is an error. The tunable :window_days
= 0 assumes clean service dates; set 1–3 only if your warehouse carries known date drift, and
expect the false-positive volume to rise sharply because genuinely repeated services (a daily
injection, twice-daily therapy) then collide. The window applies to the exact key only and is
reported as its own KEY1_DATE_DRIFT row, so KEY1_EXACT stays arithmetic; KEY2 and KEY3 remain
same-day keys. KEY2 and KEY3 are questions, not findings: cross-cover and split-component
billing are both legitimate, and the discriminating evidence is the modifier, not the count.
KEY4 is the one to run first if you have any capitated population at all - it is usually the
largest bucket and often has no rule behind it. Tune to your population; a starting point, not
a standard.

**5 · Paired control + failure mode.** CB-15 (Duplicate claim / line detection). Secondary: CB-18 (Pre-submission claim scrubber, provider side), MD-12 (PERM / MEQC sampling with error-rate measurement - duplicates are a data-processing error category).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No duplicate rule beyond exact claim resubmission. Build KEY2 and KEY4 first; they carry the exposure. |
| **Designed-not-operating** | CB-15's duplicate-hit disposition log is empty or absent for the period, so no rule fired at all. |
| **Operated-but-bypassed** | Duplicate hits were overridden at adjudication in bulk to clear a backlog, with no per-override rationale. |
| **Operated-but-ineffective** | The key is too narrow. This is the defining defect for this test and the one the four-key report is built to prove - the rule works exactly as specified, and the specification is the problem. |
| **Evidence-missing** | Hits are suppressed at the edit engine with no retained disposition log, so the rule's precision cannot be assessed at all. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 2. Medicaid NCCI - which of the six methodologies is actually firing

| | |
|---|---|
| **Paired control** | **CB-06** - Medicaid NCCI six-methodology implementation |
| **Control type** | Preventive |
| **Module** | Module 03 - Coding & Billing Integrity |
| **Owner** | Medicaid MES / fiscal agent |
| **Frequency** | Quarterly |
| **System touchpoint** | Medicaid claims (MES) |
| **Evidence artifact** | NCCI methodology attestation |
| **Control failure mode** | Unedited Medicaid claims / improper payment |
| **Regulatory anchor** | CMS Medicaid NCCI (six methodologies) [guidance] |
| **Secondary controls** | CB-04 - NCCI PTP edit enforcement pre-submission; CB-07 - MUE units-of-service edit enforcement |

**1 · Data hypothesis.** Medicaid NCCI comprises six methodologies (practitioner and outpatient-hospital PTP;
practitioner, outpatient-hospital and DME MUE; and the associated add-on code edits). A state
or plan can attest to implementation while in practice only one or two are wired into
adjudication, because the quarterly edit files are loaded for one claim type and not the
others. The signature is an edit-hit rate that is normal for one methodology and effectively
zero for another, on claim volume that should produce hits.

**2 · What to check.** Count edit hits and denials per methodology per quarter alongside the claim volume each
methodology should have been applied to. A methodology with material eligible volume and a
near-zero hit rate is not a clean population - it is an edit file that was never loaded. Cross-
check the loaded edit-file version dates against the CMS quarterly release dates.

**3 · Reference SQL.**

```sql
-- TEST 2 - Medicaid NCCI six-methodology coverage
-- Reference: CMS Medicaid NCCI, https://www.cms.gov/medicare/coding-billing/ncci-medicaid
-- Params: :period_start, :period_end, :min_eligible_lines, :min_expected_hit_rate

WITH eligible_volume AS (
  SELECT
      CASE
        WHEN cl.claim_type = 'PROFESSIONAL'  AND cl.service_category <> 'dme' THEN 'PRACTITIONER'
        WHEN cl.claim_type = 'INSTITUTIONAL' AND cl.bill_type LIKE '13%'      THEN 'OUTPATIENT_HOSPITAL'
        WHEN cl.service_category = 'dme'                                       THEN 'DME'
        ELSE 'OTHER'
      END                                     AS methodology_population,
      DATE_TRUNC('quarter', cl.service_date)  AS svc_quarter,
      COUNT(*)                                AS eligible_lines,
      SUM(cl.paid_amount)                     AS paid_amount
  FROM claim_lines cl
  WHERE cl.service_date BETWEEN :period_start AND :period_end
    AND cl.adjudication_status IN ('PAID','DENIED')
  GROUP BY 1, 2
),
edit_activity AS (
  SELECT
      eh.methodology,                 -- your edit engine's methodology tag
      eh.edit_family,                 -- 'PTP' | 'MUE' | 'ADD_ON'
      DATE_TRUNC('quarter', eh.service_date) AS svc_quarter,
      COUNT(*)                                AS edit_hits,
      SUM(CASE WHEN eh.outcome = 'DENIED' THEN 1 ELSE 0 END) AS edit_denials,
      SUM(CASE WHEN eh.outcome = 'OVERRIDDEN' THEN 1 ELSE 0 END) AS edit_overrides,
      MAX(eh.edit_file_version_date)          AS edit_file_version_used
  FROM edit_hit eh
  WHERE eh.service_date BETWEEN :period_start AND :period_end
  GROUP BY 1, 2, 3
)
SELECT
    v.svc_quarter,
    v.methodology_population,
    a.edit_family,
    v.eligible_lines,
    v.paid_amount,
    COALESCE(a.edit_hits, 0)    AS edit_hits,
    COALESCE(a.edit_denials, 0) AS edit_denials,
    COALESCE(a.edit_overrides, 0) AS edit_overrides,
    ROUND(COALESCE(a.edit_hits, 0) * 1.0 / NULLIF(v.eligible_lines, 0), 5) AS hit_rate,
    a.edit_file_version_used,
    CASE
      WHEN a.edit_hits IS NULL AND v.eligible_lines > :min_eligible_lines
        THEN 'METHODOLOGY_NOT_FIRING'
      WHEN COALESCE(a.edit_hits,0) * 1.0 / NULLIF(v.eligible_lines,0) < :min_expected_hit_rate
        THEN 'HIT_RATE_IMPLAUSIBLY_LOW'
      ELSE 'ACTIVE'
    END AS coverage_class
FROM eligible_volume v
LEFT JOIN edit_activity a
       ON a.svc_quarter = v.svc_quarter
      AND a.methodology = v.methodology_population
WHERE v.methodology_population <> 'OTHER'
  AND v.eligible_lines >= :min_eligible_lines
ORDER BY v.svc_quarter, v.methodology_population;

-- Companion, and the cheaper of the two: compare edit_file_version_used against
-- the CMS quarterly release dates. A version more than one quarter stale is a
-- finding on its own and needs no claim analysis to state.
```

**4 · Threshold guidance.** :min_expected_hit_rate has no importable value - derive it from your own history. Take each
methodology's hit rate in a quarter you have reason to trust and flag any later quarter below
roughly half of it. :min_eligible_lines = 1000 keeps thin populations out of the report. Read
the override column alongside the denial column: a high hit rate with a high override rate is
not a working edit, it is an edit being routed around, and it appears healthy on a hit-rate
metric alone. The edit-file version check needs no threshold at all and should be run first.
Tune to your population; a starting point, not a standard.

**5 · Paired control + failure mode.** CB-06 (Medicaid NCCI six-methodology implementation). Secondary: CB-04 (NCCI PTP edit enforcement pre-submission), CB-07 (MUE units-of-service edit enforcement).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | One or more methodologies were never implemented. CB-06's quarterly methodology attestation covers a state of affairs that does not exist. |
| **Designed-not-operating** | The quarterly edit files were not loaded for one or more quarters in the period - the version-date companion proves this without any claim analysis. |
| **Operated-but-bypassed** | Edits fire and are overridden at volume. Check the override rate before concluding the methodology is working. |
| **Operated-but-ineffective** | Edits are loaded for the practitioner population only and never applied to outpatient-hospital or DME claim types, so two-thirds of the methodology set is decorative. |
| **Evidence-missing** | The attestation exists but names no file version and no hit statistics, so implementation cannot be verified from the artifact. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 3. Clean-claim rate and the rework loop - the scrubber's real yield

| | |
|---|---|
| **Paired control** | **CB-18** - Pre-submission claim scrubber (provider side) |
| **Control type** | Preventive |
| **Module** | Module 03 - Coding & Billing Integrity |
| **Owner** | Revenue-cycle / billing |
| **Frequency** | Per claim |
| **System touchpoint** | Claim scrubber |
| **Evidence artifact** | Scrubber edit / clean-claim rate |
| **Control failure mode** | Dirty claims / avoidable denials |
| **Regulatory anchor** | CMS National Correct Coding Initiative (NCCI) Policy Manual [guidance]; NCCI PTP/MUE pre-payment edits [guidance]; Medicare Program Integrity Manual (Pub 100-08) [guidance] |
| **Secondary controls** | CB-04 - NCCI PTP edit enforcement pre-submission; PA-08 - Item-category documentation-eligibility checklists to prevent … |

**1 · Data hypothesis.** A pre-submission scrubber that reports a high clean-claim rate may be measuring the wrong
thing: claims that pass the scrubber and are then denied by the payer for a reason the scrubber
could have caught. The signature is a denial-reason distribution concentrated in categories the
scrubber claims to cover, and a rework loop - the same claim submitted, denied, corrected and
resubmitted - carrying real administrative cost that nobody has priced.

**2 · What to check.** Join submitted claims to their scrubber disposition and their eventual adjudication outcome.
Report the denial rate among scrubber-passed claims by denial-reason category, and separately
count resubmission chains and the days they add to the payment cycle.

**3 · Reference SQL.**

```sql
-- TEST 3 - Scrubber yield and the rework loop
-- Provider-side control, but equally runnable by a payer against its own denial stream.
-- Params: :period_start, :period_end, :min_pass_denial_rate (about 0.02, i.e. 1 in 50)

WITH submissions AS (
  SELECT c.claim_id, c.beneficiary_id, c.billing_npi,
         c.submission_date, c.original_claim_id,
         s.scrubber_outcome,            -- 'PASS' | 'EDIT_FIXED' | 'EDIT_OVERRIDDEN'
         a.adjudication_status, a.denial_reason_code, a.denial_reason_category,
         a.adjudication_date, a.paid_amount
  FROM claim c
  LEFT JOIN scrubber_result s ON s.claim_id = c.claim_id
  LEFT JOIN adjudication  a ON a.claim_id = c.claim_id
  WHERE c.submission_date BETWEEN :period_start AND :period_end
),
scrubber_yield AS (
  SELECT
      scrubber_outcome,
      denial_reason_category,
      COUNT(*) AS claims,
      SUM(CASE WHEN adjudication_status = 'DENIED' THEN 1 ELSE 0 END) AS denied
  FROM submissions
  GROUP BY scrubber_outcome, denial_reason_category
),
scrubber_yield_rated AS (
  SELECT y.*,
         SUM(y.claims) OVER (PARTITION BY y.scrubber_outcome) AS outcome_claims
  FROM scrubber_yield y
),
-- A chain is the original submission plus every resubmission that points at it, so the
-- original's own submission date starts the clock. The original is read from claim without
-- the period filter, because it may have been submitted before the period began.
-- Assumes original_claim_id always names the first submission of the chain; if your data
-- chains resubmission to resubmission, resolve the root first.
chain AS (
  SELECT COALESCE(c.original_claim_id, c.claim_id) AS chain_id,
         c.submission_date, a.adjudication_date, a.paid_amount
  FROM claim c
  LEFT JOIN adjudication a ON a.claim_id = c.claim_id
),
rework AS (
  SELECT
      chain_id,
      COUNT(*)                                  AS submissions_in_chain,  -- original included
      MIN(submission_date)                      AS first_submitted,       -- the original's date
      MAX(adjudication_date)                    AS finally_adjudicated,
      -- date minus date gives days (PostgreSQL, DuckDB, Oracle); elsewhere use DATEDIFF(day, a, b)
      MAX(adjudication_date) - MIN(submission_date) AS days_to_resolution,
      MAX(paid_amount)                          AS eventual_paid
  FROM chain
  GROUP BY chain_id
  HAVING COUNT(*) >= 3                          -- the original plus two or more resubmissions
     AND SUM(CASE WHEN submission_date BETWEEN :period_start AND :period_end
                  THEN 1 ELSE 0 END) > 0        -- at least one submission inside the period
)
SELECT 'SCRUBBER_YIELD' AS report, scrubber_outcome AS k1, denial_reason_category AS k2,
       claims AS n, denied AS m,
       ROUND(denied * 1.0 / NULLIF(outcome_claims, 0), 4) AS rate  -- denied / ALL scrubber-passed claims
FROM scrubber_yield_rated
WHERE scrubber_outcome = 'PASS' AND denied > 0
  AND denied * 1.0 / NULLIF(outcome_claims, 0) > :min_pass_denial_rate

UNION ALL

SELECT 'REWORK_LOOP', CAST(submissions_in_chain AS VARCHAR), NULL,
       COUNT(*), SUM(eventual_paid), AVG(days_to_resolution)
FROM rework
GROUP BY submissions_in_chain
ORDER BY 1, 6 DESC;

-- The number worth carrying to a governance meeting is not the clean-claim rate.
-- It is: claims that PASSED the scrubber and were then denied for a reason the
-- scrubber covers, times the fully-loaded cost of a rework cycle.
```

**4 · Threshold guidance.** There is no external benchmark for a clean-claim rate that means anything across organisations,
so do not import one. The internally meaningful metric is the scrubber-passed denial rate by
reason category, and the only threshold worth setting is relative: any category where scrubber-
passed claims are denied more often than roughly 1 in 50 is a rule the scrubber should already
have. Chains of three or more submissions are worth reading individually - they are usually one
specific edit misconfigured, not a general quality problem. Tune to your population; a starting
point, not a standard.

**5 · Paired control + failure mode.** CB-18 (Pre-submission claim scrubber, provider side). Secondary: CB-04 (NCCI PTP edit enforcement pre-submission), PA-08 (Item-category documentation-eligibility checklists).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No pre-submission scrubbing at all; the payer's edit engine is the first check any claim meets. |
| **Designed-not-operating** | The scrubber is licensed and installed but its rule set has not been updated in the period, so it enforces a stale code set. |
| **Operated-but-bypassed** | EDIT_OVERRIDDEN is a material share of submissions and no override rationale is retained - the scrubber is advisory in name only. |
| **Operated-but-ineffective** | The clean-claim rate is high because the scrubber checks format and eligibility but not coding policy, so it passes exactly the claims that will be denied. |
| **Evidence-missing** | No scrubber disposition is retained per claim, so scrubber-passed denials cannot be measured and this test cannot be run at all - which is itself the finding. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 4. PA decision timeliness - and what happens when the clock runs out

| | |
|---|---|
| **Paired control** | **PA-11** - Decision-timeliness SLA control: standard <=7 cal days; expedited <=72h (payer)/2 bus days (FFS) |
| **Control type** | Detective |
| **Module** | Module 04 - Prior Authorization |
| **Owner** | PA operations |
| **Frequency** | Continuous |
| **System touchpoint** | PA workflow timer |
| **Evidence artifact** | SLA timeliness report |
| **Control failure mode** | Untimely decisions / access-to-care harm (CMS-0057-F) |
| **Regulatory anchor** | 42 CFR 419 Subpart I (419.81-.83) [binding]; CMS-0057-F Interoperability & Prior Authorization Final Rule (2024) [binding] |
| **Secondary controls** | PA-17 - PA metrics reporting (volume, approvals/denials, average decision …; PA-12 - Specific denial-reason communication + electronic PA (FHIR API) where … |

**1 · Data hypothesis.** Where the decision-timeliness SLA control is absent or measures only the cases it closed,
prior-authorisation decisions run past the standard and expedited limits. Two distinct failures
hide behind one average: requests decided late, and requests never decided at all - still open
past the limit and therefore invisible to any metric computed over closed cases. A third
pattern is subtler and worse: a spike of denials clustered in the final hours before the clock
expires.

**2 · What to check.** Compute elapsed time from a complete request to a communicated decision, split by standard and
expedited track, and include still-open requests in the denominator. Then look at the
distribution of decisions within the allowed window - a mass of determinations in the last few
hours is a queue-management artifact, and whether those determinations skew toward denial is
the question worth asking.

**3 · Reference SQL.**

```sql
-- TEST 4 - PA decision timeliness, including never-decided requests
-- Anchors: 42 CFR 419 Subpart I; CMS-0057-F. Confirm the limits that bind YOUR line
-- of business before setting parameters - they differ by programme and have changed.
-- Params: :period_start, :period_end, :standard_limit_days (7), :expedited_limit_hours (72)

WITH requests AS (
  SELECT pa.pa_request_id, pa.beneficiary_id, pa.requesting_npi, pa.service_code,
         pa.track,                       -- 'STANDARD' | 'EXPEDITED'
         pa.received_datetime,
         pa.complete_datetime,           -- when the package became complete
         pa.decision_datetime,
         pa.decision,                    -- 'AFFIRMED' | 'NON_AFFIRMED' | 'PARTIAL' | NULL if open
         pa.notice_sent_datetime
  FROM pa_request pa
  WHERE pa.received_datetime BETWEEN :period_start AND :period_end
),
timed AS (
  SELECT r.*,
         COALESCE(r.decision_datetime, CURRENT_TIMESTAMP) AS clock_stop,
         EXTRACT(EPOCH FROM (COALESCE(r.decision_datetime, CURRENT_TIMESTAMP)
                             - COALESCE(r.complete_datetime, r.received_datetime))) / 3600.0
           AS elapsed_hours,
         CASE WHEN r.track = 'EXPEDITED' THEN :expedited_limit_hours
              ELSE :standard_limit_days * 24 END AS limit_hours
  FROM requests r
)
SELECT
    track,
    CASE
      WHEN decision IS NULL AND elapsed_hours > limit_hours THEN 'OPEN_PAST_LIMIT'
      WHEN decision IS NULL                                  THEN 'OPEN_WITHIN_LIMIT'
      WHEN elapsed_hours > limit_hours                       THEN 'DECIDED_LATE'
      WHEN elapsed_hours > limit_hours * 0.9                 THEN 'DECIDED_IN_FINAL_TENTH'
      ELSE 'DECIDED_ON_TIME'
    END AS timeliness_class,
    COUNT(*)                                                        AS requests,
    ROUND(AVG(elapsed_hours), 1)                                    AS mean_elapsed_hours,
    SUM(CASE WHEN decision = 'NON_AFFIRMED' THEN 1 ELSE 0 END)      AS non_affirmed,
    ROUND(SUM(CASE WHEN decision = 'NON_AFFIRMED' THEN 1.0 ELSE 0 END)
          / NULLIF(COUNT(*), 0), 4)                                 AS non_affirm_rate,
    -- notice lag is a separate obligation from the decision itself
    ROUND(AVG(EXTRACT(EPOCH FROM (notice_sent_datetime - decision_datetime)) / 3600.0), 1)
          AS mean_notice_lag_hours
FROM timed
GROUP BY 1, 2
ORDER BY track, timeliness_class;

-- Read DECIDED_IN_FINAL_TENTH's non_affirm_rate against DECIDED_ON_TIME's.
-- A materially higher denial rate in the last tenth of the window is a queue
-- artifact, not a clinical pattern, and it is a control finding about the
-- workflow rather than about any individual determination.
```

**4 · Threshold guidance.** The limits are regulatory, not tunable - but they differ by programme, line of business and
state, and CMS-0057-F changed several of them, so read the rule that binds you rather than the
defaults above. What is tunable is the final-tenth window: 0.9 is a starting cut, and the
comparison that matters is between denial rates inside and outside it, not the raw count.
OPEN_PAST_LIMIT should be zero and any non-zero count is a finding; it is also the number most
likely to be missing from an existing timeliness report, because reports built over closed
cases cannot see it. Tune to your population; a starting point, not a standard.

**5 · Paired control + failure mode.** PA-11 (Decision-timeliness SLA control). Secondary: PA-17 (PA metrics reporting), PA-12 (Specific denial-reason communication + electronic PA).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No SLA timer exists on the PA workflow, so elapsed time cannot be computed from the system of record at all. |
| **Designed-not-operating** | PA-11 specifies continuous SLA monitoring; no timeliness report was produced for the period. |
| **Operated-but-bypassed** | The clock is restarted by issuing a documentation request, and requests are issued routinely rather than because the package is incomplete. Check the rate of documentation requests against the incompleteness the package actually shows. |
| **Operated-but-ineffective** | The report is computed over closed cases only, so requests that were never decided - the worst outcome for the beneficiary - are structurally invisible to it. |
| **Evidence-missing** | Decision timestamps are overwritten on status change rather than appended, so elapsed time cannot be reconstructed for any closed case. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 5. Denial notices that cannot be appealed - reason specificity and channel

| | |
|---|---|
| **Paired control** | **PA-12** - Specific denial-reason communication + electronic PA (FHIR API) where required |
| **Control type** | Preventive |
| **Module** | Module 04 - Prior Authorization |
| **Owner** | PA operations |
| **Frequency** | Per denial |
| **System touchpoint** | PA portal / FHIR PA API |
| **Evidence artifact** | Denial notice with specific reason |
| **Control failure mode** | Vague denials that impede appeal (CMS-0057-F) |
| **Regulatory anchor** | CMS-0057-F Interoperability & Prior Authorization Final Rule (2024) [binding] |
| **Secondary controls** | PA-11 - Decision-timeliness SLA control: standard <=7 cal days; expedited …; PA-14 - PA denial-appropriateness sampling review [open OIG rec] |

**1 · Data hypothesis.** A denial communicated with a generic reason code is functionally unappealable: the requester
cannot tell what to correct or what evidence to add. Where the specific-denial-reason control
is absent, denial-reason distributions collapse onto a handful of catch-all codes, and appeal
rates on those codes fall - not because the denials are correct, but because nobody can
construct an appeal against them.

**2 · What to check.** Rank denial-reason codes by volume and compute, per code, the appeal rate and the overturn rate
on appeal. A high-volume code with a low appeal rate and a high overturn rate among the few
that are appealed is the specific signature: the denials are wrong and the notice is preventing
challenge. Report electronic-PA channel adoption alongside it, because a notice that never
reaches the requester's system is the same failure.

**3 · Reference SQL.**

```sql
-- TEST 5 - Denial-reason specificity and appealability
-- Params: :period_start, :period_end, :min_denials (50)

WITH denials AS (
  SELECT d.pa_request_id, d.denial_reason_code, d.denial_reason_text,
         d.notice_channel,              -- 'FHIR_API' | 'PORTAL' | 'FAX' | 'MAIL'
         d.decision_datetime, d.requesting_npi
  FROM pa_decision d
  WHERE d.decision = 'NON_AFFIRMED'
    AND d.decision_datetime BETWEEN :period_start AND :period_end
),
appeals AS (
  SELECT a.pa_request_id,
         MAX(CASE WHEN a.appeal_outcome = 'OVERTURNED' THEN 1 ELSE 0 END) AS overturned,
         COUNT(*) AS appeal_levels
  FROM pa_appeal a
  GROUP BY a.pa_request_id
)
SELECT
    d.denial_reason_code,
    MIN(d.denial_reason_text)                        AS example_text,
    LENGTH(MIN(d.denial_reason_text))                AS reason_text_length,
    COUNT(*)                                         AS denials,
    COUNT(DISTINCT d.requesting_npi)                 AS distinct_requesters,
    SUM(CASE WHEN ap.pa_request_id IS NOT NULL THEN 1 ELSE 0 END) AS appealed,
    ROUND(SUM(CASE WHEN ap.pa_request_id IS NOT NULL THEN 1.0 ELSE 0 END)
          / NULLIF(COUNT(*), 0), 4)                  AS appeal_rate,
    ROUND(SUM(COALESCE(ap.overturned, 0)) * 1.0
          / NULLIF(SUM(CASE WHEN ap.pa_request_id IS NOT NULL THEN 1 ELSE 0 END), 0), 4)
                                                     AS overturn_rate_when_appealed,
    ROUND(SUM(CASE WHEN d.notice_channel = 'FHIR_API' THEN 1.0 ELSE 0 END)
          / NULLIF(COUNT(*), 0), 4)                  AS electronic_notice_share
FROM denials d
LEFT JOIN appeals ap ON ap.pa_request_id = d.pa_request_id
GROUP BY d.denial_reason_code
HAVING COUNT(*) >= :min_denials
ORDER BY denials DESC;

-- The cell to read first: a high-volume code with a LOW appeal_rate and a HIGH
-- overturn_rate_when_appealed. That combination means the few requesters who did
-- appeal usually won, and the rest could not tell what to argue.
```

**4 · Threshold guidance.** No absolute threshold applies - the metric is comparative across your own reason codes. The
pattern to escalate is a code in the top decile by volume whose appeal rate sits in the bottom
quartile and whose overturn rate when appealed sits in the top quartile. reason_text_length is
a crude but useful proxy for specificity and costs nothing to compute: a modal reason text
under about 80 characters across thousands of denials is almost certainly a template. Low
electronic_notice_share is a finding in its own right where the electronic-PA requirement
binds. Tune to your population; a starting point, not a standard.

**5 · Paired control + failure mode.** PA-12 (Specific denial-reason communication + electronic PA / FHIR API where required). Secondary: PA-11 (Decision-timeliness SLA), PA-14 (PA denial-appropriateness sampling).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | Denials carry a code but no case-specific reason text; PA-12's denial notice with specific reason does not exist as an artifact. |
| **Designed-not-operating** | The notice template supports a specific reason field and it is left empty in practice. |
| **Operated-but-bypassed** | Reviewers select a catch-all code to close cases faster; concentration on one or two codes across many reviewers is the tell. |
| **Operated-but-ineffective** | The notice is specific about the rule applied but never states what evidence would change the outcome, so it is specific and still unappealable. |
| **Evidence-missing** | Notices are generated but not retained in a form tied to the request, so specificity cannot be assessed retrospectively. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 6. Denial appropriateness - overturn rate as the outcome metric the sample should have found

| | |
|---|---|
| **Paired control** | **PA-14** - PA denial-appropriateness sampling review [open OIG rec] |
| **Control type** | Detective |
| **Module** | Module 04 - Prior Authorization |
| **Owner** | Compliance / Medical director |
| **Frequency** | Quarterly |
| **System touchpoint** | PA system + audit tool |
| **Evidence artifact** | Denial-sample review workpaper |
| **Control failure mode** | Systemic inappropriate denials undetected (OIG OEI-09-19-00350) |
| **Regulatory anchor** | OIG OEI-09-19-00350 (17 Jul 2023) - Medicaid MC PA denials; rec 23-E-09-029.01 open [case evidence] |
| **Secondary controls** | PA-15 - Automatic external medical review of upheld denials (managed care) …; PA-09 - Coverage-rule application: apply current NCD/LCD/LCA; internal … |

**1 · Data hypothesis.** Where denial-appropriateness sampling is absent or samples without regard to outcome,
systemically inappropriate denials persist. The payer already holds the outcome measure: the
rate at which its own denials are overturned on appeal. A high overturn rate is not evidence
that appeals work; it is evidence that the original determinations were wrong, and it is
strongest exactly where the appeal rate is low, because then the overturns are a sample of a
much larger untested population.

**2 · What to check.** Compute overturn rate by service category, by reviewer, and by determination basis, and
extrapolate the implied incorrect-denial count over the un-appealed population as a range
rather than a point estimate. Then check whether PA-14's quarterly sample was drawn at all, and
whether it was drawn from the strata this analysis says are worst.

**3 · Reference SQL.**

```sql
-- TEST 6 - Denial appropriateness via overturn rate, and sample-coverage check
-- Anchor: OIG OEI-09-19-00350 (open recommendation).
-- Params: :period_start, :period_end, :min_denials (30; the floor for material denial volume)

WITH denials AS (
  SELECT d.pa_request_id, d.service_category, d.reviewer_id, d.determination_basis,
         d.decision_datetime
  FROM pa_decision d
  WHERE d.decision = 'NON_AFFIRMED'
    AND d.decision_datetime BETWEEN :period_start AND :period_end
),
outcomes AS (
  SELECT a.pa_request_id,
         MAX(CASE WHEN a.appeal_outcome = 'OVERTURNED' THEN 1 ELSE 0 END) AS overturned
  FROM pa_appeal a GROUP BY a.pa_request_id
),
strata AS (
  SELECT
      d.service_category,
      COUNT(*)                                                     AS denials,
      SUM(CASE WHEN o.pa_request_id IS NOT NULL THEN 1 ELSE 0 END) AS appealed,
      SUM(COALESCE(o.overturned, 0))                               AS overturned,
      ROUND(SUM(COALESCE(o.overturned,0)) * 1.0
            / NULLIF(SUM(CASE WHEN o.pa_request_id IS NOT NULL THEN 1 ELSE 0 END),0), 4)
                                                                   AS overturn_rate
  FROM denials d
  LEFT JOIN outcomes o ON o.pa_request_id = d.pa_request_id
  GROUP BY d.service_category
)
SELECT
    s.*,
    (s.denials - s.appealed)                                       AS never_appealed,
    -- Illustrative range only. Appealed cases are a self-selected sample and are
    -- NOT representative of the un-appealed population. Report as a range, label
    -- it as an ordering aid, and never as an overpayment or liability estimate.
    ROUND((s.denials - s.appealed) * s.overturn_rate * 0.25)       AS implied_wrong_low,
    ROUND((s.denials - s.appealed) * s.overturn_rate)              AS implied_wrong_high,
    smp.sampled_cases,
    CASE WHEN smp.sampled_cases IS NULL THEN 'STRATUM_NEVER_SAMPLED' ELSE 'SAMPLED' END
      AS pa14_coverage
FROM strata s
LEFT JOIN (
    SELECT service_category, COUNT(*) AS sampled_cases
    FROM pa_denial_sample
    WHERE sample_date BETWEEN :period_start AND :period_end
    GROUP BY service_category
) smp ON smp.service_category = s.service_category
WHERE s.denials >= :min_denials          -- material strata only; smaller strata are left out of the report
ORDER BY s.overturn_rate DESC NULLS LAST, s.denials DESC;

-- Also run this grouped by reviewer_id. A reviewer whose overturn rate sits far
-- above peers on comparable case mix is a training and calibration finding, not
-- a disciplinary one, and the output should say so on its face.
```

**4 · Threshold guidance.** Overturn rate has no defensible external benchmark, and importing one would be worse than
useless here because case mix drives it. Compare strata against each other and against the same
stratum's own history. Material denial volume is set by :min_denials (30 denials in the period
is a starting value); strata below it are left out of the report, so lower it if small
categories matter to you. Any stratum at or above that floor and STRATUM_NEVER_SAMPLED is a
finding regardless of its overturn rate, because it means PA-14's quarterly sample is not risk-
weighted. The extrapolation columns are deliberately a wide range and are an ordering aid only
 -  appealed cases are self-selected and systematically unrepresentative, and a point estimate
built on them will not survive scrutiny. Tune to your population; a starting point, not a
standard.

**5 · Paired control + failure mode.** PA-14 (PA denial-appropriateness sampling review - open OIG recommendation). Secondary: PA-15 (Automatic external medical review of upheld denials), PA-09 (Coverage-rule application; internal criteria not more restrictive than Medicare rules).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No denial-appropriateness sampling exists. PA-14's denial-sample review workpaper does not exist; the open OIG recommendation is unaddressed. |
| **Designed-not-operating** | PA-14 specifies a quarterly cycle; one or more quarters produced no workpaper. |
| **Operated-but-bypassed** | Samples are drawn and reviewed by the same function that issued the determinations, so the review confirms rather than tests. |
| **Operated-but-ineffective** | The sample is drawn at random across all denials, so it is dominated by high-volume low-risk categories and structurally under-samples the strata this test identifies as worst. |
| **Evidence-missing** | Reviews are performed but dispositions are not retained per sampled case, so the error rate cannot be recomputed or audited. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 7. Upheld denials that never reached an external reviewer

| | |
|---|---|
| **Paired control** | **PA-15** - Automatic external medical review of upheld denials (managed care) [open OIG rec] |
| **Control type** | Corrective |
| **Module** | Module 04 - Prior Authorization |
| **Owner** | Compliance |
| **Frequency** | Per upheld denial |
| **System touchpoint** | External-review referral |
| **Evidence artifact** | External-review disposition |
| **Control failure mode** | Inappropriate denials not independently checked (OIG) |
| **Regulatory anchor** | OIG OEI-09-19-00350 (17 Jul 2023) - Medicaid MC PA denials; rec 23-E-09-029.04 open [case evidence] |
| **Secondary controls** | PA-14 - PA denial-appropriateness sampling review [open OIG rec]; PA-19 - Qualified-clinician medical-necessity denial governance + segregation … |

**1 · Data hypothesis.** Where automatic external medical review of upheld denials is absent, an internal reviewer's
decision to uphold their own organisation's denial is final. The data signature is direct:
upheld internal appeals with no matching external-review referral record. A second, quieter
signature is a referral that exists but resolves suspiciously fast or always in the plan's
favour, which suggests the external review is not independent in practice.

**2 · What to check.** Left-join upheld internal appeals to the external-review referral log and count the unreferred.
Then, for the referred population, report time-to-disposition and outcome distribution by
external entity - an entity that never overturns is a finding about the referral arrangement.

**3 · Reference SQL.**

```sql
-- TEST 7 - Upheld denials with no external medical review
-- Anchor: OIG OEI-09-19-00350 (open recommendation), managed-care context.
-- Params: :period_start, :period_end, :referral_window_days (administrative lag in days, e.g. 7 to 30; must be shorter than the period)

WITH upheld AS (
  SELECT a.appeal_id, a.pa_request_id, a.beneficiary_id, a.service_category,
         a.appeal_level, a.disposition_date
  FROM pa_appeal a
  WHERE a.appeal_outcome = 'UPHELD'
    AND a.disposition_date BETWEEN :period_start AND :period_end
    -- administrative lag: an appeal upheld inside the last :referral_window_days of the period
    -- has not yet had time to be referred and is not treated as unreferred
    -- date minus integer days (PostgreSQL, DuckDB, Oracle); elsewhere use DATEADD(day, -n, end_date)
    AND a.disposition_date <= CAST(:period_end AS DATE) - CAST(:referral_window_days AS INTEGER)
    AND a.appeal_level = (SELECT MAX(x.appeal_level) FROM pa_appeal x
                          WHERE x.pa_request_id = a.pa_request_id)
),
referred AS (
  SELECT e.appeal_id, e.external_entity_id, e.referral_date,
         e.external_disposition, e.external_disposition_date
  FROM external_review e
)
SELECT
    CASE WHEN r.appeal_id IS NULL THEN 'NOT_REFERRED' ELSE 'REFERRED' END AS referral_class,
    u.service_category,
    r.external_entity_id,
    COUNT(*) AS appeals,
    ROUND(AVG(r.referral_date - u.disposition_date), 1)               AS mean_days_to_referral,
    ROUND(AVG(r.external_disposition_date - r.referral_date), 1)      AS mean_days_to_external_decision,
    SUM(CASE WHEN r.external_disposition = 'OVERTURNED' THEN 1 ELSE 0 END) AS externally_overturned,
    ROUND(SUM(CASE WHEN r.external_disposition = 'OVERTURNED' THEN 1.0 ELSE 0 END)
          / NULLIF(COUNT(*), 0), 4)                                   AS external_overturn_rate
FROM upheld u
LEFT JOIN referred r ON r.appeal_id = u.appeal_id
GROUP BY 1, 2, 3
ORDER BY referral_class, appeals DESC;

-- Two findings, deliberately in one report:
--   NOT_REFERRED with material volume  -> the control is absent or partial.
--   REFERRED with an external_overturn_rate of effectively zero across an entity
--   with high volume -> the referral exists and the independence does not.
```

**4 · Threshold guidance.** NOT_REFERRED is categorical where the automatic-referral obligation applies - any count above
zero is a finding, and the only tuning needed is :referral_window_days to allow for
administrative lag before treating a case as unreferred. For the referred population there is
no threshold, only a comparison: an external entity whose overturn rate is near zero while
another entity reviewing comparable case mix overturns at a normal rate is the pattern to
escalate, and with a single external entity you have no comparator at all - which is itself
worth recording. Tune to your population; a starting point, not a standard.

**5 · Paired control + failure mode.** PA-15 (Automatic external medical review of upheld denials - open OIG recommendation). Secondary: PA-14 (Denial-appropriateness sampling), PA-19 (Qualified-clinician denial governance + segregation of duties).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No external-review referral route exists; PA-15's external-review disposition artifact does not exist. |
| **Designed-not-operating** | The referral route exists on paper and no referrals were made in the period. |
| **Operated-but-bypassed** | Cases are closed as withdrawn or reopened internally rather than referred, keeping the referral count low without changing the outcome for the beneficiary. |
| **Operated-but-ineffective** | Referrals are made to an entity selected and paid by the plan with no independence safeguard, and the near-zero overturn rate shows it. |
| **Evidence-missing** | Referrals are asserted but no dated referral record or external disposition is retained per appeal. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 8. Denials with no human clinician behind them

| | |
|---|---|
| **Paired control** | **PA-19** - Qualified-clinician medical-necessity denial governance + segregation of duties (no denial solely by algorithm) |
| **Control type** | Preventive |
| **Module** | Module 04 - Prior Authorization |
| **Owner** | Medical director / Compliance |
| **Frequency** | Per denial |
| **System touchpoint** | Clinical review platform |
| **Evidence artifact** | Clinician sign-off + SoD matrix |
| **Control failure mode** | Improper automated denials (42 CFR 422.101(c)) |
| **Regulatory anchor** | 42 CFR 422.566(d) (a physician or other appropriate health care professional with relevant expertise must review before a partly or fully adverse medical-necessity organization determination is issued; Medicare Advantage) [binding]; 42 CFR 422.101(c)(1)(i)(C)-(D) (medical-necessity decisions rest on the enrollee's history, physician recommendations, clinical notes and, where appropriate, medical director involvement) [binding]; related, not support for clinician review: CMS-0057-F, 89 FR 8758 (8 Feb 2024), which requires a specific denial reason and prior-authorization decision timeframes only [binding] |
| **Secondary controls** | PA-12 - Specific denial-reason communication + electronic PA (FHIR API) where …; PA-14 - PA denial-appropriateness sampling review [open OIG rec] |

**1 · Data hypothesis.** Medical-necessity denials require a qualified clinician's determination and segregation of
duties; no denial may be issued solely by an algorithm. Where that governance is absent or
nominal, the signature is arithmetic rather than statistical: a reviewer whose sign-off count
per working hour exceeds what reading a case permits, sign-offs clustered in seconds-apart
bursts, or a sign-off identity that matches the identity that ran the model.

**2 · What to check.** Count medical-necessity denials with no clinician identifier at all; then, for those with one,
compute per-reviewer sign-offs per active hour and the distribution of inter-signature
intervals. Separately check the credential of the signing identity against the determination
type, and check that the signer is not the requester or the model operator.

**3 · Reference SQL.**

```sql
-- TEST 8 - Clinician sign-off integrity on medical-necessity denials
-- Anchor: 42 CFR 422.101(c); CMS-0057-F.
-- Params: :period_start, :period_end, :min_plausible_review_seconds (60), :max_denials_per_day (100), :max_fast_signoffs (10)

WITH mn_denials AS (
  SELECT d.pa_request_id, d.decision_datetime, d.reviewer_id, d.reviewer_credential,
         d.determination_basis,         -- 'CLINICAL_REVIEW' | 'AUTOMATED_CRITERIA' | 'MODEL'
         d.model_id, d.model_version,
         d.service_category
  FROM pa_decision d
  WHERE d.decision = 'NON_AFFIRMED'
    AND d.denial_type = 'MEDICAL_NECESSITY'
    AND d.decision_datetime BETWEEN :period_start AND :period_end
),
sequenced AS (
  SELECT m.*,
         LAG(m.decision_datetime) OVER (PARTITION BY m.reviewer_id
                                        ORDER BY m.decision_datetime) AS prev_signoff
  FROM mn_denials m
),
per_reviewer AS (
  SELECT
      reviewer_id,
      -- a reviewer with no credential on file is reported as UNRECORDED, not skipped
      COALESCE(MIN(reviewer_credential), 'UNRECORDED') AS credential,
      COUNT(*)                                     AS denials_signed,
      COUNT(DISTINCT CAST(decision_datetime AS DATE)) AS active_days,
      ROUND(AVG(EXTRACT(EPOCH FROM (decision_datetime - prev_signoff))), 1)
                                                   AS mean_seconds_between_signoffs,
      SUM(CASE WHEN EXTRACT(EPOCH FROM (decision_datetime - prev_signoff))
                    < :min_plausible_review_seconds THEN 1 ELSE 0 END)
                                                   AS signoffs_faster_than_plausible
  FROM sequenced
  WHERE reviewer_id IS NOT NULL                    -- denials with no reviewer are counted separately below
  GROUP BY reviewer_id
)
SELECT 'NO_REVIEWER_RECORDED' AS finding, CAST(NULL AS VARCHAR) AS reviewer_id,
       COUNT(*) AS n, CAST(NULL AS VARCHAR) AS detail
FROM mn_denials WHERE reviewer_id IS NULL

UNION ALL

SELECT 'MODEL_BASIS_NO_CLINICIAN', CAST(NULL AS VARCHAR), COUNT(*), MIN(model_id)
FROM mn_denials
WHERE determination_basis IN ('AUTOMATED_CRITERIA','MODEL') AND reviewer_id IS NULL

UNION ALL

SELECT 'IMPLAUSIBLE_SIGNOFF_RATE', reviewer_id, denials_signed,
       CAST(signoffs_faster_than_plausible AS VARCHAR)
FROM per_reviewer
WHERE denials_signed * 1.0 / NULLIF(active_days, 0) > :max_denials_per_day
   OR signoffs_faster_than_plausible > :max_fast_signoffs

UNION ALL

SELECT 'CREDENTIAL_MISMATCH', reviewer_id, denials_signed, credential
FROM per_reviewer
WHERE credential NOT IN ('MD','DO','NP','PA','RN_WITH_MD_OVERSIGHT')  -- adapt to your policy; UNRECORDED is listed
ORDER BY 1, 3 DESC;
```

**4 · Threshold guidance.** NO_REVIEWER_RECORDED and MODEL_BASIS_NO_CLINICIAN are categorical - any count above zero is a
finding and needs no tuning. :min_plausible_review_seconds and :max_denials_per_day are the
tunable pair and both are genuinely contestable: a reviewer working a queue of near-identical
straightforward cases can legitimately move fast, and a low threshold here will produce
indefensible output. Start deliberately high - 60 seconds and 100 per day - and note that the
defensible finding is a burst pattern (many sign-offs seconds apart) rather than a daily total.
:max_fast_signoffs (10 as a starting value) is how many faster-than-plausible sign-offs a
reviewer may have before being listed. A reviewer with no credential on file is listed under
CREDENTIAL_MISMATCH as UNRECORDED. Nothing in this test identifies an individual's misconduct;
it identifies a workflow that does not permit the review it certifies. Tune to your population;
a starting point, not a standard.

**5 · Paired control + failure mode.** PA-19 (Qualified-clinician medical-necessity denial governance + segregation of duties; no denial solely by algorithm). Secondary: PA-12 (Specific denial-reason communication), PA-14 (Denial-appropriateness sampling).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No clinician sign-off is captured on medical-necessity denials; PA-19's sign-off + SoD matrix does not exist. |
| **Designed-not-operating** | The sign-off field exists and is null for a material share of denials in the period. |
| **Operated-but-bypassed** | A single clinician identity is applied in bulk to model output, so the sign-off is a batch attestation rather than a determination. |
| **Operated-but-ineffective** | Sign-off is captured but segregation of duties is not enforced - the signer is the requester, the model operator, or the person with the utilisation target. |
| **Evidence-missing** | Sign-offs are recorded in a field that is overwritten on reassignment, so the person who made each determination cannot be established after the fact. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 9. Internal criteria more restrictive than the coverage rule they implement

| | |
|---|---|
| **Paired control** | **PA-09** - Coverage-rule application: apply current NCD/LCD/LCA; internal criteria not more restrictive than Medicare rules |
| **Control type** | Preventive |
| **Module** | Module 04 - Prior Authorization |
| **Owner** | Medical reviewer / Medical director |
| **Frequency** | Per review |
| **System touchpoint** | Coverage-rules engine |
| **Evidence artifact** | Criteria-to-coverage crosswalk |
| **Control failure mode** | Improper denial via over-restrictive plan criteria (OIG MAO) |
| **Regulatory anchor** | SSA 1862(a)(1)(A) [binding]; OIG OEI-09-18-00260 (27 Apr 2022) - MAO PA denials; all three recommendations closed [case evidence] |
| **Secondary controls** | PA-14 - PA denial-appropriateness sampling review [open OIG rec]; PA-16 - EPSDT / childrens-services PA policy control (Medicaid) [open GAO rec] |

**1 · Data hypothesis.** Coverage determinations must apply the current national and local coverage rules, and internal
criteria may not be more restrictive than them. Where that control is absent, a plan's internal
criteria set drifts stricter - usually not by decision but by accretion, as a criteria vendor's
rule set is adopted wholesale. The signature is a denial population whose stated basis is an
internal criterion for a service the national rule covers, and a denial rate for that service
far above comparable payers or above the plan's own rate before the criteria change.

**2 · What to check.** Group denials by the criterion cited and join to the coverage-rule crosswalk that PA-09 names
as its evidence artifact. Flag every denial whose cited basis is an internal criterion with no
mapped NCD/LCD/LCA, and every service where the internal criterion's effective date is followed
by a step change in denial rate.

**3 · Reference SQL.**

```sql
-- TEST 9 - Internal criteria vs the coverage rule
-- Anchors: SSA 1862(a)(1)(A); OIG OEI-09-18-00260 (open rec).
-- Params: :period_start, :period_end, :min_determinations (100)

WITH cited AS (
  SELECT d.pa_request_id, d.service_code, d.criterion_id, d.decision, d.decision_datetime
  FROM pa_decision d
  WHERE d.decision_datetime BETWEEN :period_start AND :period_end
),
mapped AS (
  SELECT c.*, x.ncd_lcd_id, x.rule_effective_date, x.criterion_effective_date,
         x.mapping_status              -- 'MAPPED' | 'INTERNAL_ONLY' | 'UNMAPPED'
  FROM cited c
  LEFT JOIN criteria_coverage_crosswalk x ON x.criterion_id = c.criterion_id
),
rates AS (
  SELECT service_code, criterion_id, ncd_lcd_id, mapping_status,
         MIN(criterion_effective_date) AS criterion_effective_date,
         DATE_TRUNC('month', decision_datetime) AS decision_month,
         COUNT(*) AS determinations,
         SUM(CASE WHEN decision = 'NON_AFFIRMED' THEN 1 ELSE 0 END) AS denials
  FROM mapped
  GROUP BY 1,2,3,4,6
)
SELECT
    service_code, criterion_id, ncd_lcd_id, mapping_status,
    criterion_effective_date, decision_month,
    determinations, denials,
    ROUND(denials * 1.0 / NULLIF(determinations, 0), 4) AS denial_rate,
    -- denial rate for the same service in the six months before the criterion took effect
    LAG(ROUND(denials * 1.0 / NULLIF(determinations,0), 4), 6)
      OVER (PARTITION BY service_code ORDER BY decision_month) AS denial_rate_6m_prior,
    CASE
      WHEN mapping_status = 'INTERNAL_ONLY' THEN 'NO_COVERAGE_RULE_BEHIND_CRITERION'
      WHEN mapping_status = 'UNMAPPED' OR mapping_status IS NULL THEN 'CROSSWALK_INCOMPLETE'  -- NULL: criterion cited but absent from the crosswalk
      ELSE 'MAPPED'
    END AS criterion_class
FROM rates
WHERE determinations >= :min_determinations
ORDER BY criterion_class, denial_rate DESC;

-- CROSSWALK_INCOMPLETE is not a lesser finding than NO_COVERAGE_RULE_BEHIND_CRITERION.
-- It means PA-09's evidence artifact - the criteria-to-coverage crosswalk - does not
-- cover the criteria actually in use, so the control cannot be assessed at all.
```

**4 · Threshold guidance.** NO_COVERAGE_RULE_BEHIND_CRITERION is categorical in the programmes where the not-more-
restrictive obligation binds, and needs no threshold; whether it binds depends on your line of
business, so establish that before treating a row as a finding. The step-change comparison is
the tunable half: a denial-rate rise of more than roughly a third across a criterion's
effective date, on stable volume, is worth reading - but coding changes, seasonal mix and
network changes all move denial rates, so treat it as a prompt to read the criterion text
rather than as evidence. :min_determinations = 100 keeps the rate meaningful. Tune to your
population; a starting point, not a standard.

**5 · Paired control + failure mode.** PA-09 (Coverage-rule application: apply current NCD/LCD/LCA; internal criteria not more restrictive than Medicare rules - open OIG recommendation). Secondary: PA-14 (Denial-appropriateness sampling), PA-16 (EPSDT / children's-services PA policy).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No criteria-to-coverage crosswalk exists; PA-09's evidence artifact is missing and internal criteria are unmapped to any coverage rule. |
| **Designed-not-operating** | The crosswalk exists but was not refreshed against the current NCD/LCD/LCA releases in the period, so it maps to superseded rules. |
| **Operated-but-bypassed** | Reviewers cite the internal criterion in preference to the coverage rule because the workflow surfaces it first. |
| **Operated-but-ineffective** | The crosswalk maps criteria to rules but nobody compares their restrictiveness, so the mapping exists and the substantive test it was built for is never performed. |
| **Evidence-missing** | The criterion cited on each determination is not retained, so no determination can be traced to the rule it applied. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 10. Determinations made on incomplete utilisation data

| | |
|---|---|
| **Paired control** | **PA-10** - Utilization-data sufficiency check before coverage determinations |
| **Control type** | Detective |
| **Module** | Module 04 - Prior Authorization |
| **Owner** | Medical review / Data |
| **Frequency** | Per determination |
| **System touchpoint** | Claims + utilization datamart |
| **Evidence artifact** | Data-completeness attestation |
| **Control failure mode** | Coverage decision made on incomplete data (OIG 2025) |
| **Regulatory anchor** | OIG OEI-BL-19-00501 (20 Feb 2025) - Part B coverage determinations utilization data; data rec 25-E-BL-015.01 closed, .02 (guidance on home and professional settings) open [case evidence] |
| **Secondary controls** | PA-03 - Complete PA package assembly & validation before submission (order + …; PA-09 - Coverage-rule application: apply current NCD/LCD/LCA; internal … |

**1 · Data hypothesis.** A coverage determination made without the beneficiary's utilisation history is a determination
made blind - it cannot see the prior conservative therapy that would support approval, or the
duplicate item already supplied that would support denial. Where the data-sufficiency check is
absent, determinations proceed on whatever happened to be loaded. The signature is a
determination timestamp preceding the availability of the beneficiary's claims history, or a
data-completeness attestation that is present and always identical.

**2 · What to check.** For each determination, establish whether the beneficiary's utilisation datamart contained a
full lookback window at the moment of decision, and whether the data-completeness attestation
was recorded. Report determinations made against a partial or empty history, split by whether
the outcome was affirmation or denial - both directions are harmful and for different reasons.

**3 · Reference SQL.**

```sql
-- TEST 10 - Utilisation-data sufficiency at the moment of determination
-- Anchor: OIG (2025) Part B coverage determinations utilisation data (open rec).
-- Params: :period_start, :period_end, :lookback_days (365)

WITH determinations AS (
  SELECT d.pa_request_id, d.beneficiary_id, d.service_code, d.decision,
         d.decision_datetime, d.data_completeness_attested
  FROM pa_decision d
  WHERE d.decision_datetime BETWEEN :period_start AND :period_end
),
history_at_decision AS (
  SELECT
      t.pa_request_id,
      -- claims present in the datamart AS AT the decision, not as at today.
      -- The warehouse_load_date filter is the whole point of this test: a claim
      -- loaded after the decision could not have informed it.
      -- History is NOT cut at the lookback: the earliest line available must be
      -- allowed to lie beyond it, or a full lookback could never be recognised.
      COUNT(cl.claim_line_id) AS lines_available_at_decision,       -- all history loaded before the decision
      -- the same count restricted to the lookback window, which is what the reviewer could use.
      -- Date minus integer days (PostgreSQL, DuckDB, Oracle); elsewhere use DATEADD(day, -n, d).
      COUNT(CASE WHEN cl.service_date >= CAST(t.decision_datetime AS DATE) - :lookback_days
                 THEN cl.claim_line_id END) AS lines_in_lookback_at_decision,
      MIN(cl.service_date)    AS earliest_available_service_date,
      MAX(cl.warehouse_load_date) AS latest_load_before_decision
  FROM determinations t
  LEFT JOIN claim_lines cl
         ON  cl.beneficiary_id     = t.beneficiary_id
         AND cl.warehouse_load_date <  t.decision_datetime
  GROUP BY t.pa_request_id
)
SELECT
    t.decision,
    CASE
      WHEN h.lines_available_at_decision = 0 THEN 'NO_HISTORY_AVAILABLE'
      -- date minus date gives days (PostgreSQL, DuckDB, Oracle); elsewhere use DATEDIFF(day, a, b)
      WHEN CAST(t.decision_datetime AS DATE) - h.earliest_available_service_date < :lookback_days
        THEN 'PARTIAL_LOOKBACK'
      ELSE 'FULL_LOOKBACK'
    END AS data_sufficiency_class,
    COALESCE(t.data_completeness_attested, 'NOT_RECORDED') AS attestation,
    COUNT(*) AS determinations,
    COUNT(DISTINCT t.service_code) AS distinct_services,
    SUM(h.lines_available_at_decision)   AS history_lines_all_time,
    SUM(h.lines_in_lookback_at_decision) AS history_lines_in_lookback
FROM determinations t
JOIN history_at_decision h ON h.pa_request_id = t.pa_request_id
GROUP BY 1, 2, 3
ORDER BY 1, 2;

-- Read both directions. NO_HISTORY_AVAILABLE + NON_AFFIRMED is an access finding:
-- the prior therapy that would have supported approval may simply not have been
-- visible. NO_HISTORY_AVAILABLE + AFFIRMED is a payment finding: the duplicate
-- already supplied was equally invisible. Reporting only one of them is the
-- single most common way this analysis gets used badly.
```

**4 · Threshold guidance.** :lookback_days = 365 is a generic starting window and the clinically right window varies by
service - a conservative-therapy requirement may look back six weeks, a replacement-interval
rule five years. Set it per service class rather than globally, or the output is dominated by
an arbitrary choice. The class is set by the earliest line ever loaded;
history_lines_in_lookback counts only the lines inside the window, so a FULL_LOOKBACK row with
few lines in the window is a long-enrolled member with little recent utilisation, not a data
gap. NO_HISTORY_AVAILABLE is categorical and always worth reading. An attestation value of
NOT_RECORDED across a material share of determinations means PA-10's evidence artifact is not
being produced; an attestation that is present and identical on every determination means it is
being produced without being performed, which is harder to see and no better. Tune to your
population; a starting point, not a standard.

**5 · Paired control + failure mode.** PA-10 (Utilization-data sufficiency check before coverage determinations - open OIG recommendation). Secondary: PA-03 (Complete PA package assembly & validation), PA-09 (Coverage-rule application).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No data-sufficiency check precedes determinations; PA-10's data-completeness attestation does not exist. |
| **Designed-not-operating** | The check is specified per determination and is absent from a material share of them. |
| **Operated-but-bypassed** | The attestation is defaulted to a passing value in the workflow, so it is recorded on every determination regardless of the underlying data. |
| **Operated-but-ineffective** | The check confirms that the datamart responded, not that it contained a meaningful lookback - an empty result and a complete history both pass. |
| **Evidence-missing** | Warehouse load dates are not retained, so what data was available at the moment of decision cannot be reconstructed and this test cannot be run. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 11. Insufficient-documentation non-affirmation - whose package, whose checklist

| | |
|---|---|
| **Paired control** | **PA-03** - Complete PA package assembly & validation before submission (order + supporting medical records) |
| **Control type** | Preventive |
| **Module** | Module 04 - Prior Authorization |
| **Owner** | PA coordinator |
| **Frequency** | Per request |
| **System touchpoint** | Document mgmt / PA portal |
| **Evidence artifact** | PA package checklist |
| **Control failure mode** | Insufficient-documentation non-affirmation |
| **Regulatory anchor** | 42 CFR 410.38 [binding]; CMS PIM 100-08 (Program Integrity Manual) [guidance] |
| **Secondary controls** | PA-08 - Item-category documentation-eligibility checklists to prevent …; PA-12 - Specific denial-reason communication + electronic PA (FHIR API) where … |

**1 · Data hypothesis.** Non-affirmation for insufficient documentation is the most common PA outcome that nobody owns.
Where package assembly and validation is absent on the requester side, it concentrates in
specific requesters; where the payer's documentation-eligibility checklist is absent or
unclear, it concentrates in specific service categories across all requesters. The two have
opposite remedies, and the concentration pattern distinguishes them.

**2 · What to check.** Compute the insufficient-documentation non-affirmation rate by requester and by service
category, and measure the dispersion within each. A category where the rate is high and similar
across most requesters is a payer-side clarity problem; a category where the rate is normal for
most and extreme for a few is a requester-side assembly problem.

**3 · Reference SQL.**

```sql
-- TEST 11 - Insufficient-documentation non-affirmation: requester vs category
-- Params: :period_start, :period_end, :min_requests (30), :category_wide_threshold (0.20), :low_dispersion (0.05), :requester_z (2.5)

WITH reqs AS (
  SELECT p.pa_request_id, p.requesting_npi, p.service_category,
         p.decision, p.denial_reason_category, p.resubmitted_flag,
         p.decision_datetime
  FROM pa_decision p
  WHERE p.decision_datetime BETWEEN :period_start AND :period_end
),
by_requester AS (
  SELECT service_category, requesting_npi,
         COUNT(*) AS requests,
         SUM(CASE WHEN decision = 'NON_AFFIRMED'
                   AND denial_reason_category = 'INSUFFICIENT_DOCUMENTATION'
                  THEN 1 ELSE 0 END) AS insuf_denials,
         SUM(CASE WHEN resubmitted_flag = 1 THEN 1 ELSE 0 END) AS resubmitted
  FROM reqs
  GROUP BY service_category, requesting_npi
  HAVING COUNT(*) >= :min_requests
),
category_shape AS (
  SELECT service_category,
         COUNT(*)                                   AS requesters,
         AVG(insuf_denials * 1.0 / requests)        AS mean_insuf_rate,
         STDDEV(insuf_denials * 1.0 / requests)     AS sd_insuf_rate
  FROM by_requester
  GROUP BY service_category
)
SELECT
    b.service_category, b.requesting_npi, b.requests, b.insuf_denials,
    ROUND(b.insuf_denials * 1.0 / b.requests, 4) AS insuf_rate,
    ROUND(c.mean_insuf_rate, 4)                  AS category_mean,
    ROUND(c.sd_insuf_rate, 4)                    AS category_sd,
    ROUND((b.insuf_denials * 1.0 / b.requests - c.mean_insuf_rate)
          / NULLIF(c.sd_insuf_rate, 0), 2)       AS z_within_category,
    ROUND(b.resubmitted * 1.0 / b.requests, 4)   AS resubmission_rate,
    CASE
      WHEN c.mean_insuf_rate > :category_wide_threshold AND c.sd_insuf_rate < :low_dispersion
        THEN 'PAYER_SIDE_CLARITY_PROBLEM'
      WHEN (b.insuf_denials * 1.0 / b.requests - c.mean_insuf_rate)
           / NULLIF(c.sd_insuf_rate, 0) > :requester_z
        THEN 'REQUESTER_SIDE_ASSEMBLY_PROBLEM'
      ELSE 'WITHIN_NORM'
    END AS diagnosis
FROM by_requester b
JOIN category_shape c ON c.service_category = b.service_category
ORDER BY diagnosis, insuf_rate DESC;
```

**4 · Threshold guidance.** :category_wide_threshold = 0.20 and :low_dispersion = 0.05 together define 'high for everyone',
which is the payer-side reading; :requester_z = 2.5 defines the requester-side outlier. All
three are starting values and the classification is only as good as the peer grouping behind
it, so build service categories that are clinically comparable before trusting the diagnosis
column. Read the resubmission rate alongside: a high insufficient-documentation rate with a
high resubmission rate and eventual approval is administrative friction, which is a real cost
but not an integrity finding, and it should not be reported as one. Tune to your population; a
starting point, not a standard.

**5 · Paired control + failure mode.** PA-03 (Complete PA package assembly & validation before submission). Secondary: PA-08 (Item-category documentation-eligibility checklists - open OIG recommendation), PA-12 (Specific denial-reason communication).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No package checklist is applied before submission; PA-03's per-request checklist artifact does not exist. |
| **Designed-not-operating** | The checklist exists and is not applied - a completion rate far below the request count proves it. |
| **Operated-but-bypassed** | The checklist is completed as a formality after submission rather than as a gate before it. |
| **Operated-but-ineffective** | The checklist lists document types but not the content each must contain, so a complete package still fails on sufficiency. |
| **Evidence-missing** | Denial reason categories are too coarse to isolate insufficient documentation, so the rate cannot be computed and the control cannot be tested. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 12. High-denial categories with no documentation-eligibility checklist

| | |
|---|---|
| **Paired control** | **PA-08** - Item-category documentation-eligibility checklists to prevent insufficient-documentation denials [open OIG rec] |
| **Control type** | Preventive |
| **Module** | Module 04 - Prior Authorization |
| **Owner** | PA coordinator |
| **Frequency** | Per request |
| **System touchpoint** | PA portal |
| **Evidence artifact** | Category eligibility checklist |
| **Control failure mode** | Denials/overpayments for documentation gaps (OIG catheters) |
| **Regulatory anchor** | OIG A-09-22-03019 (4 Feb 2025) - intermittent urinary catheters; rec 25-A-09-042.02 open, .01 and .03 closed [case evidence] |
| **Secondary controls** | PA-03 - Complete PA package assembly & validation before submission (order + …; CD-05 - Item/service-category documentation-eligibility checklist to prevent … |

**1 · Data hypothesis.** Where item-category documentation-eligibility checklists are absent, the categories that need
them most are identifiable from the payer's own denial data - and the absence is usually not a
decision but an omission that nobody has ranked. The signature is a category with high denial
volume, high insufficient-documentation share, and no checklist entry in the checklist
inventory.

**2 · What to check.** Rank service categories by insufficient-documentation denial volume and dollars, left-join the
checklist inventory, and report the ranked categories with no checklist. This is a coverage
report, not a detection query: its output is a prioritised build list for the PA function.

**3 · Reference SQL.**

```sql
-- TEST 12 - Checklist coverage against denial concentration
-- Anchor: OIG (2025) intermittent urinary catheters (open rec) - the exemplar case.
-- Params: :period_start, :period_end, :min_insuf_denials (100)

WITH denial_load AS (
  SELECT d.service_category,
         COUNT(*) AS determinations,
         SUM(CASE WHEN d.decision = 'NON_AFFIRMED' THEN 1 ELSE 0 END) AS denials,
         SUM(CASE WHEN d.decision = 'NON_AFFIRMED'
                   AND d.denial_reason_category = 'INSUFFICIENT_DOCUMENTATION'
                  THEN 1 ELSE 0 END) AS insuf_denials,
         SUM(CASE WHEN d.decision = 'NON_AFFIRMED' THEN d.requested_amount ELSE 0 END)
                                     AS denied_dollars
  FROM pa_decision d
  WHERE d.decision_datetime BETWEEN :period_start AND :period_end
  GROUP BY d.service_category
)
SELECT
    dl.service_category,
    dl.determinations, dl.denials, dl.insuf_denials, dl.denied_dollars,
    ROUND(dl.insuf_denials * 1.0 / NULLIF(dl.denials, 0), 4) AS insuf_share_of_denials,
    cli.checklist_id,
    cli.last_reviewed_date,
    CASE
      WHEN cli.checklist_id IS NULL THEN 'NO_CHECKLIST'
      WHEN cli.last_reviewed_date < :period_start - 365 THEN 'CHECKLIST_STALE'
      ELSE 'CURRENT'
    END AS checklist_class
FROM denial_load dl
LEFT JOIN checklist_inventory cli ON cli.service_category = dl.service_category
WHERE dl.insuf_denials >= :min_insuf_denials
ORDER BY checklist_class, dl.insuf_denials DESC;

-- Output is a build list. Rank by insuf_denials rather than denied_dollars:
-- the checklist saves a rework cycle per denial, so volume is the right unit,
-- and the dollar column is there to break ties.
```

**4 · Threshold guidance.** :min_insuf_denials = 100 keeps the list short enough to act on; lower it once the top of the
list is built. There is no statistical threshold here at all - the report is a join against an
inventory, and NO_CHECKLIST on a high-volume category is a finding on its face. CHECKLIST_STALE
deserves the same weight as NO_CHECKLIST when the underlying coverage policy has changed in the
interval, so check the policy dates before deprioritising it. Tune to your population; a
starting point, not a standard.

**5 · Paired control + failure mode.** PA-08 (Item-category documentation-eligibility checklists to prevent insufficient-documentation denials - open OIG recommendation). Secondary: PA-03 (Complete PA package assembly & validation), CD-05 (documentation-checklist control).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No checklist inventory exists at all, so this query cannot be joined - which is the finding, and it takes one look at the schema to establish. |
| **Designed-not-operating** | Checklists exist for a handful of categories and have not been extended to the categories the denial data has been pointing at for years. |
| **Operated-but-bypassed** | Checklists exist and are not surfaced in the request workflow, so requesters never see them. |
| **Operated-but-ineffective** | Checklists are written from the coverage rule's language rather than from the actual denial reasons, so following one does not prevent the denial. |
| **Evidence-missing** | No inventory records which categories have checklists or when they were last reviewed, so coverage cannot be assessed. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 13. EPSDT and children's services - PA as a barrier

| | |
|---|---|
| **Paired control** | **PA-16** - EPSDT / childrens-services PA policy control (Medicaid) [open GAO rec] |
| **Control type** | Preventive |
| **Module** | Module 04 - Prior Authorization |
| **Owner** | Medical policy |
| **Frequency** | Annual |
| **System touchpoint** | Policy repository |
| **Evidence artifact** | EPSDT PA policy statement |
| **Control failure mode** | Improper PA barriers to EPSDT services (GAO-24-106532) |
| **Regulatory anchor** | GAO-24-106532 (30 Apr 2024) - Medicaid MC PA for children; Recs 1 and 2 to CMS, open [case evidence] |
| **Secondary controls** | PA-14 - PA denial-appropriateness sampling review [open OIG rec]; PA-11 - Decision-timeliness SLA control: standard <=7 cal days; expedited … |

**1 · Data hypothesis.** EPSDT obliges Medicaid to cover services necessary to correct or ameliorate a condition for
enrollees under 21, on a standard broader than adult medical necessity. Where the EPSDT PA
policy control is absent, adult criteria are applied to paediatric requests by default  - 
usually because one criteria set is configured for all ages. The signature is a denial rate for
under-21 enrollees at or above the adult rate on the same service, and denials citing adult-
configured criteria.

**2 · What to check.** Compare determination outcomes for under-21 and 21-plus enrollees on the same service codes,
and check whether the criterion cited on paediatric denials is EPSDT-scoped. Report separately
the paediatric denials citing a criterion with no paediatric variant at all - that population
is the control failure stated most directly.

**3 · Reference SQL.**

```sql
-- TEST 13 - EPSDT / paediatric PA outcomes vs adult
-- Anchor: GAO-24-106532 (open rec); EPSDT under SSA 1905(r).
-- Params: :period_start, :period_end, :min_paediatric (30)

WITH d AS (
  SELECT p.pa_request_id, p.service_code, p.criterion_id, p.decision, p.decision_datetime,
         CASE WHEN b.age_at_service < 21 THEN 'UNDER_21' ELSE 'ADULT' END AS age_band
  FROM pa_decision p
  JOIN beneficiary b ON b.beneficiary_id = p.beneficiary_id
  WHERE p.decision_datetime BETWEEN :period_start AND :period_end
),
rates AS (
  SELECT service_code, age_band,
         COUNT(*) AS determinations,
         SUM(CASE WHEN decision = 'NON_AFFIRMED' THEN 1 ELSE 0 END) AS denials
  FROM d GROUP BY service_code, age_band
),
paired AS (
  SELECT p.service_code,
         p.determinations AS paed_determinations, p.denials AS paed_denials,
         a.determinations AS adult_determinations, a.denials AS adult_denials,
         ROUND(p.denials * 1.0 / NULLIF(p.determinations,0), 4) AS paed_denial_rate,
         ROUND(a.denials * 1.0 / NULLIF(a.determinations,0), 4) AS adult_denial_rate
  FROM rates p
  JOIN rates a ON a.service_code = p.service_code AND a.age_band = 'ADULT'
  WHERE p.age_band = 'UNDER_21'
)
SELECT
    pr.*,
    ROUND(pr.paed_denial_rate - pr.adult_denial_rate, 4) AS rate_gap,
    COALESCE(cx.epsdt_variant_exists, 0) AS epsdt_variant_exists,
    CASE
      WHEN COALESCE(cx.epsdt_variant_exists, 0) = 0 THEN 'ADULT_CRITERION_APPLIED_TO_CHILD'
      WHEN pr.paed_denial_rate > pr.adult_denial_rate THEN 'PAEDIATRIC_RATE_EXCEEDS_ADULT'
      ELSE 'WITHIN_NORM'
    END AS epsdt_class
FROM paired pr
LEFT JOIN (
    SELECT DISTINCT d.service_code,
           MAX(CASE WHEN c.epsdt_scoped = 1 THEN 1 ELSE 0 END) AS epsdt_variant_exists
    FROM d LEFT JOIN criteria_master c ON c.criterion_id = d.criterion_id   -- unknown criterion = no EPSDT variant
    WHERE d.age_band = 'UNDER_21'
    GROUP BY d.service_code
) cx ON cx.service_code = pr.service_code
WHERE pr.paed_determinations >= :min_paediatric
ORDER BY epsdt_class, rate_gap DESC;
```

**4 · Threshold guidance.** ADULT_CRITERION_APPLIED_TO_CHILD is categorical where EPSDT applies and is the row to act on;
it requires no rate comparison at all, only the criteria-master join. The rate gap is the
softer signal and needs care: paediatric and adult case mix differ on the same code, so a
positive gap is a prompt to read the criterion rather than evidence of a barrier.
:min_paediatric = 30 keeps thin cells out. This is a beneficiary-protection test and its output
must never be used to justify tightening anything - the only direction of remediation it
supports is toward the EPSDT standard. Tune to your population; a starting point, not a
standard.

**5 · Paired control + failure mode.** PA-16 (EPSDT / children's-services PA policy control - open GAO recommendation). Secondary: PA-14 (Denial-appropriateness sampling), PA-11 (Decision-timeliness SLA).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No EPSDT-specific PA policy exists; PA-16's annual EPSDT PA policy statement does not exist and one criteria set serves all ages. |
| **Designed-not-operating** | The policy statement exists and was not reviewed in the annual cycle the control specifies. |
| **Operated-but-bypassed** | EPSDT variants exist in the criteria master and are not selected by the workflow because age is not a routing attribute. |
| **Operated-but-ineffective** | The EPSDT variant restates the adult criterion with a paediatric label, so the broader correct-or-ameliorate standard is never actually applied. |
| **Evidence-missing** | Age at service is not retained on the determination record, so paediatric outcomes cannot be separated at all. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 14. PA metrics reporting - recompute the published numbers

| | |
|---|---|
| **Paired control** | **PA-17** - PA metrics reporting (volume, approvals/denials, average decision timeframe) |
| **Control type** | Detective |
| **Module** | Module 04 - Prior Authorization |
| **Owner** | PA operations / Compliance |
| **Frequency** | Annual |
| **System touchpoint** | PA analytics / reporting |
| **Evidence artifact** | Published PA metrics report |
| **Control failure mode** | No transparency into PA performance (CMS-0057-F) |
| **Regulatory anchor** | CMS-0057-F Interoperability & Prior Authorization Final Rule (2024) [binding] |
| **Secondary controls** | PA-11 - Decision-timeliness SLA control: standard <=7 cal days; expedited …; PA-14 - PA denial-appropriateness sampling review [open OIG rec] |

**1 · Data hypothesis.** Published PA metrics - volume, approval and denial rates, average decision timeframe - are
computed by a reporting layer that is rarely reconciled to the transactional record. Where the
reporting control is absent or unverified, the published numbers drift from the system of
record in a consistent direction: withdrawn and abandoned requests dropped from the
denominator, never-decided requests excluded from the timeframe average, partial approvals
counted as approvals.

**2 · What to check.** Recompute each published metric from the transactional PA tables using an explicit, documented
denominator, and place the recomputed value next to the published value. The output is a
reconciliation, and every difference should be explainable by a stated definitional choice - an
unexplainable difference is the finding.

**3 · Reference SQL.**

```sql
-- TEST 14 - Recompute published PA metrics from the transactional record
-- Anchor: CMS-0057-F public reporting. Params: :period_start, :period_end

WITH base AS (
  SELECT r.pa_request_id, r.track, r.received_datetime, r.decision, r.decision_datetime,
         r.request_status          -- 'DECIDED' | 'WITHDRAWN' | 'ABANDONED' | 'OPEN'
  FROM pa_request r
  WHERE r.received_datetime BETWEEN :period_start AND :period_end
),
recomputed AS (
  SELECT
      COUNT(*)                                                          AS requests_received,
      SUM(CASE WHEN request_status = 'DECIDED' THEN 1 ELSE 0 END)       AS decided,
      SUM(CASE WHEN request_status IN ('WITHDRAWN','ABANDONED') THEN 1 ELSE 0 END)
                                                                        AS withdrawn_abandoned,
      SUM(CASE WHEN request_status = 'OPEN' THEN 1 ELSE 0 END)          AS still_open,
      SUM(CASE WHEN decision = 'AFFIRMED' THEN 1 ELSE 0 END)            AS affirmed,
      SUM(CASE WHEN decision = 'PARTIAL'  THEN 1 ELSE 0 END)            AS partial,
      SUM(CASE WHEN decision = 'NON_AFFIRMED' THEN 1 ELSE 0 END)        AS non_affirmed,
      -- denominator stated explicitly: all requests received, not just those decided
      ROUND(SUM(CASE WHEN decision = 'NON_AFFIRMED' THEN 1.0 ELSE 0 END)
            / NULLIF(COUNT(*), 0), 4)                                   AS denial_rate_all_received,
      ROUND(SUM(CASE WHEN decision = 'NON_AFFIRMED' THEN 1.0 ELSE 0 END)
            / NULLIF(SUM(CASE WHEN request_status = 'DECIDED' THEN 1 ELSE 0 END), 0), 4)
                                                                        AS denial_rate_decided_only,
      ROUND(AVG(CASE WHEN decision_datetime IS NOT NULL
                     THEN EXTRACT(EPOCH FROM (decision_datetime - received_datetime))/86400.0
                END), 2)                                                AS mean_days_decided_only
  FROM base
)
SELECT r.*, p.published_denial_rate, p.published_mean_days, p.published_request_count,
       ROUND(r.denial_rate_decided_only - p.published_denial_rate, 4) AS denial_rate_delta,
       ROUND(r.mean_days_decided_only  - p.published_mean_days, 2)    AS mean_days_delta,
       (r.requests_received - p.published_request_count)              AS request_count_delta
FROM recomputed r
CROSS JOIN published_pa_metrics p
WHERE p.reporting_period_start = :period_start;

-- The two denial-rate variants are printed side by side on purpose. Which one is
-- correct is a definitional question, not a data question - but a published figure
-- that matches neither, or that matches the flattering one with no stated
-- definition, is a reporting-control finding.
```

**4 · Threshold guidance.** No threshold. This is a reconciliation and the standard is exact agreement given a stated
definition. The tolerance worth allowing is timing - a report run on a different cut date will
differ slightly - so reconcile on the same as-at date before treating a delta as real. The
delta that matters most is request_count_delta: a published denominator smaller than the
received count means requests are being dropped somewhere in the reporting layer, and until
that is explained none of the rates above it can be read. Tune to your population; a starting
point, not a standard.

**5 · Paired control + failure mode.** PA-17 (PA metrics reporting - volume, approvals/denials, average decision timeframe). Secondary: PA-11 (Decision-timeliness SLA), PA-14 (Denial-appropriateness sampling).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No PA metrics are published or produced; PA-17's published metrics report does not exist. |
| **Designed-not-operating** | The report is specified annually and was not produced for the period. |
| **Operated-but-bypassed** | Figures are adjusted between computation and publication with no change record. |
| **Operated-but-ineffective** | The metric definitions exclude withdrawn, abandoned and never-decided requests, so the published rates describe a population chosen after the fact rather than the requests actually received. |
| **Evidence-missing** | No definition document accompanies the published figures, so they cannot be reconciled to anything and this test's deltas cannot be adjudicated. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 15. PERM data-processing error surface - rebuild it before the cycle does

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
| **Secondary controls** | CB-15 - Duplicate claim / line detection; CB-06 - Medicaid NCCI six-methodology implementation |

**1 · Data hypothesis.** PERM separates data-processing errors - the payment was wrong because of how the claim was
processed, independent of whether the service was appropriate - from medical-review and
eligibility errors. The data-processing component is the one a payer can rebuild for itself
between cycles, because every input is internal: pricing, duplicates, third-party liability,
unit and rate application, non-covered codes. Where PERM/MEQC measurement is the only error-
rate signal, a state learns its data-processing error rate once a cycle, years after the
payments.

**2 · What to check.** Assemble a continuous internal proxy for the data-processing error categories: repriced amount
versus paid amount, duplicates from test 1, TPL-liable claims paid as primary, units above the
authorised or edited maximum, and payments on codes not covered on the date of service. Report
each category as a rate against total paid, and reconcile the total against the last published
PERM data-processing rate.

**3 · Reference SQL.**

```sql
-- TEST 15 - Continuous internal proxy for the PERM data-processing error component
-- Anchor: 42 CFR Part 431 Subpart Q (PERM/MEQC).
-- Params: :period_start, :period_end, :pricing_tolerance (0.01)

WITH paid AS (
  SELECT cl.claim_line_id, cl.beneficiary_id, cl.hcpcs, cl.units, cl.service_date,
         cl.paid_amount, cl.allowed_amount, cl.pricing_source
  FROM claim_lines cl
  WHERE cl.service_date BETWEEN :period_start AND :period_end
    AND cl.adjudication_status = 'PAID'
),
pricing_variance AS (
  SELECT p.claim_line_id, p.paid_amount,
         fs.fee_schedule_amount * p.units AS expected_amount,
         p.paid_amount - (fs.fee_schedule_amount * p.units) AS variance
  FROM paid p
  JOIN fee_schedule fs
    ON  fs.hcpcs = p.hcpcs
    AND p.service_date BETWEEN fs.effective_start AND COALESCE(fs.effective_end, DATE '9999-12-31')
  WHERE ABS(p.paid_amount - (fs.fee_schedule_amount * p.units)) > :pricing_tolerance
),
tpl_paid_primary AS (
  SELECT p.claim_line_id, p.paid_amount
  FROM paid p
  JOIN tpl_coverage t
    ON  t.beneficiary_id = p.beneficiary_id
    AND p.service_date BETWEEN t.coverage_start AND COALESCE(t.coverage_end, DATE '9999-12-31')
  WHERE p.pricing_source <> 'SECONDARY'
),
noncovered_on_dos AS (
  SELECT p.claim_line_id, p.paid_amount
  FROM paid p
  LEFT JOIN covered_code cc
    ON  cc.hcpcs = p.hcpcs
    AND p.service_date BETWEEN cc.coverage_start AND COALESCE(cc.coverage_end, DATE '9999-12-31')
  WHERE cc.hcpcs IS NULL
),
totals AS (SELECT SUM(paid_amount) AS total_paid, COUNT(*) AS total_lines FROM paid)
SELECT 'PRICING_VARIANCE'  AS dp_category, COUNT(*) AS lines, SUM(ABS(variance))    AS dollars FROM pricing_variance
UNION ALL
SELECT 'TPL_PAID_PRIMARY',  COUNT(*), SUM(paid_amount) FROM tpl_paid_primary
UNION ALL
SELECT 'NONCOVERED_ON_DOS', COUNT(*), SUM(paid_amount) FROM noncovered_on_dos
UNION ALL
SELECT 'TOTAL_PAID_BASE',   total_lines, total_paid FROM totals;

-- Add the duplicate categories from TEST 1 as further rows rather than recomputing
-- them here, so the two tests cannot disagree. Express each category as a share of
-- TOTAL_PAID_BASE and compare the sum to your last published PERM data-processing
-- rate. A proxy far BELOW the PERM rate means your categories are incomplete, not
-- that your payments are cleaner than PERM found them.
```

**4 · Threshold guidance.** :pricing_tolerance = $0.01 catches genuine repricing differences; raise it to a dollar or two
if your fee schedule carries known rounding, but do not raise it far, because a systematic
small variance across millions of lines is exactly the error PERM is built to find and a
generous tolerance hides it. There is no target error rate here - the comparison is against
your own last PERM data-processing rate and against the previous period of this same proxy.
Treat a large gap in either direction as a question about the proxy first and about the
payments second, because an incomplete category list is the more likely explanation. Tune to
your population; a starting point, not a standard.

**5 · Paired control + failure mode.** MD-12 (PERM / MEQC eligibility & claims sampling with error-rate measurement). Secondary: CB-15 (Duplicate claim / line detection), CB-06 (Medicaid NCCI six-methodology implementation).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No error-rate measurement exists outside the federal cycle; MD-12's sampled-case workpapers and error rate exist only when CMS requires them. |
| **Designed-not-operating** | The cycle ran and the corrective-action plan it produced was not implemented, so the same categories recur. |
| **Operated-but-bypassed** | Sampled cases are corrected individually and the underlying processing defect is not fixed, so the error rate is managed rather than reduced. |
| **Operated-but-ineffective** | Measurement is cycle-bound only. Years of payments accumulate between measurements and nothing internal reports the rate in the interval - which is the specific gap this test closes. |
| **Evidence-missing** | Sampling workpapers do not retain the error category per case, so the data-processing component cannot be separated from medical-review and eligibility errors. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 16. PTP edits that fire and are overridden - the bypass rate nobody reports

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
| **Secondary controls** | CB-05 - Modifier-59 / X{EPSU} governance & documentation review; CB-17 - Prepayment edit maintenance & tuning from emerging vulnerabilities |

**1 · Data hypothesis.** An NCCI PTP edit that fires and is then overridden produces the same payment as an edit that
never fired, but it looks healthy on every hit-rate metric. Where edit-enforcement governance
is absent, override authority is broad, override reasons are not required, and the override
rate concentrates in a small number of operators or a small number of code pairs. The dollars
behind overrides are the edit set's real leakage and are almost never reported.

**2 · What to check.** Compute, per PTP code pair and per overriding operator, the hit count, override count, override
rate and the dollars released by override. Then check whether an override reason was recorded
and whether the modifier that would have legitimately bypassed the edit was actually present on
the claim.

**3 · Reference SQL.**

```sql
-- TEST 16 - PTP edit override rate and the dollars it releases
-- Reference: CMS NCCI PTP edits & Policy Manual.
-- Params: :period_start, :period_end, :min_hits (25)

WITH hits AS (
  SELECT eh.edit_hit_id, eh.claim_line_id, eh.code_pair_id,
         eh.column1_code, eh.column2_code, eh.modifier_indicator,
         eh.outcome,                    -- 'DENIED' | 'OVERRIDDEN' | 'PAID_WITH_MODIFIER'
         eh.override_operator_id, eh.override_reason_code,
         cl.modifier_1, cl.modifier_2, cl.paid_amount, cl.rendering_npi
  FROM edit_hit eh
  JOIN claim_lines cl ON cl.claim_line_id = eh.claim_line_id
  WHERE eh.edit_family = 'PTP'
    AND eh.service_date BETWEEN :period_start AND :period_end
)
SELECT
    code_pair_id, column1_code, column2_code, modifier_indicator,
    COUNT(*)                                                          AS hits,
    SUM(CASE WHEN outcome = 'OVERRIDDEN' THEN 1 ELSE 0 END)           AS overrides,
    ROUND(SUM(CASE WHEN outcome = 'OVERRIDDEN' THEN 1.0 ELSE 0 END)
          / NULLIF(COUNT(*), 0), 4)                                   AS override_rate,
    SUM(CASE WHEN outcome = 'OVERRIDDEN' THEN paid_amount ELSE 0 END) AS dollars_released,
    COUNT(DISTINCT override_operator_id)                              AS distinct_operators,
    SUM(CASE WHEN outcome = 'OVERRIDDEN' AND override_reason_code IS NULL THEN 1 ELSE 0 END)
                                                                      AS overrides_no_reason,
    -- an override on a pair whose modifier indicator is '0' can never be legitimate:
    -- indicator 0 means no modifier permits the pair to be billed together.
    SUM(CASE WHEN outcome = 'OVERRIDDEN' AND modifier_indicator = '0' THEN 1 ELSE 0 END)
                                                                      AS overrides_on_indicator_0,
    SUM(CASE WHEN outcome = 'OVERRIDDEN'
              AND modifier_indicator = '1'
              AND COALESCE(modifier_1,'') NOT IN ('59','XE','XS','XP','XU')
              AND COALESCE(modifier_2,'') NOT IN ('59','XE','XS','XP','XU')
             THEN 1 ELSE 0 END)                                       AS overrides_without_modifier
FROM hits
GROUP BY code_pair_id, column1_code, column2_code, modifier_indicator
HAVING COUNT(*) >= :min_hits
    -- categorical: any override on an indicator-0 pair is reported whatever the volume
    OR SUM(CASE WHEN outcome = 'OVERRIDDEN' AND modifier_indicator = '0' THEN 1 ELSE 0 END) > 0
ORDER BY dollars_released DESC;

-- Run the same aggregation grouped by override_operator_id. A single operator
-- carrying a large share of overrides is a workflow and authority finding - the
-- question is what queue pressure produced it, not who the person is.
```

**4 · Threshold guidance.** overrides_on_indicator_0 is categorical: a modifier indicator of 0 means no modifier permits
the pair, so an override there cannot be correct and any count is a finding. Confirm the
indicator values against the current NCCI files before relying on this, since indicators change
between quarterly releases. override_rate has no external benchmark - compare pairs against
each other and against the same pair's own history, and read dollars_released rather than the
rate when deciding what to act on. :min_hits = 25 keeps the list stable. Tune to your
population; a starting point, not a standard.

**5 · Paired control + failure mode.** CB-04 (NCCI PTP edit enforcement pre-submission). Secondary: CB-05 (Modifier-59 / X{EPSU} governance & documentation review), CB-17 (Prepayment edit maintenance & tuning from emerging vulnerabilities).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | PTP edits are not enforced; CB-04's PTP edit hit / disposition log does not exist. |
| **Designed-not-operating** | Edits are loaded and set to informational, so they log hits and stop nothing. |
| **Operated-but-bypassed** | This is the test's central case: edits fire and are overridden at volume, with overrides_no_reason establishing that the bypass is not even documented. |
| **Operated-but-ineffective** | Override authority is granted broadly to resolve claim backlogs, so the edit's effect depends on queue pressure rather than on coding policy. |
| **Evidence-missing** | Overrides are applied without recording an operator or reason, so the bypass rate can be counted but never adjudicated. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## Gap notes - controls this family needs that the library does not yet have

*Each note names a control objective these tests point to that the control library does not yet state. They are candidates for a future library release; the library itself is unchanged by this pack.*

### G1. No control owns the duplicate KEY itself, only the existence of duplicate detection.

CB-15 requires duplicate claim/line detection and names a duplicate-hit disposition log as its
evidence. It says nothing about what the duplicate key must contain. Test 1 shows that the key
is where the control actually succeeds or fails: a rule keyed on billing NPI can never see a
cross-NPI duplicate, and a rule confined to the FFS system can never see a service already
covered by capitation. An assessor can confirm CB-15 affirmatively - the rule exists, the log
exists, hits are dispositioned - while the largest duplicate population in the programme is
structurally invisible to it. This is the same shape as family 7's G1: the control names a
technique and leaves the parameter that determines its effectiveness unstated.

**Recommendation.** Extend CB-15 to state the minimum key set the duplicate rule must cover - exact; cross-
rendering-NPI within a TIN; professional/institutional pairing; and FFS against capitated
benefit - with a periodic re-test of each. Stating the keys is what makes the control
assessable; stating only that detection exists is not.

### G2. No control governs override authority on prepayment edits.

CB-04, CB-06 and CB-07 each require edits to be enforced, and CB-05 governs the modifiers used
to bypass them. None addresses the operator-side override - who may release an edit, on what
basis, whether a reason must be recorded, and whether the override rate is reported anywhere.
Test 16 shows the override path produces exactly the same payment as an absent edit while
presenting as a working control on every hit-rate metric, and that overrides on a modifier-
indicator-0 pair cannot be legitimate under any reading. The dollars released by override are,
in most payers, not reported to anyone.

**Recommendation.** An edit-override governance control for Module 03: named override authority with segregation
from claim submission, a mandatory reason code, a prohibition on override where the modifier
indicator forbids the pair, and periodic reporting of override rate and dollars released by
code pair and by operator. This pairs naturally with CB-17's edit-change board, which is the
body that should receive the report.

### G3. The beneficiary-protection direction now has controls but no measurement obligation.

Family 4's G2 recorded that no control addressed improper denial for eligibility decisions, and
noted PA-14 and PA-15 as the templates. This family shows the templates working: eleven of its
sixteen tests surface improper denial or delay, and PA-11, PA-12, PA-14, PA-15, PA-16 and PA-19
all exist to be paired to. What none of them states is an obligation to measure the outcome.
PA-14 requires sampling but not that the sample be risk-weighted by overturn rate; PA-15
requires referral but not that external overturn rates be monitored; PA-17 requires metrics
reporting but does not require the denial-rate denominator to be defined. Tests 6, 7 and 14
each had to invent the outcome measure the control should already have named.

**Recommendation.** Add an outcome-measurement requirement to the existing PA controls rather than authoring a new
one - overturn rate by stratum feeding PA-14's sample design, external overturn rate by entity
under PA-15, and a stated denominator under PA-17. This is a smaller change than a new control
and it is what converts six existing beneficiary-protection controls from procedural to
testable.

### G4. Module 04 is now fully covered by analytics, and that exposes a scoping question rather than closing one.

With this family, all nineteen Module 04 controls carry a query pattern. But the module is
titled Prior Authorization and now carries controls that are not about prior authorisation at
all: PA-19 is a governance and segregation-of-duties control that applies to any medical-
necessity denial, and PA-10's data-sufficiency requirement applies to coverage determinations
generally. Both were paired here because Module 04 is where they live, but an assessor scoping
a prior-authorisation assessment will under-apply them, and an assessor scoping a claims or
appeals assessment will not find them at all. This is the CB-05 naming problem from family 2 at
module scale.

**Recommendation.** No control change proposed. Record in the crosswalk that PA-10 and PA-19 apply to coverage
determinations and medical-necessity denials generally, not only to prior authorisation, so
they surface in both scopes. If the library is ever reorganised, these two are candidates for a
determination-governance grouping that spans Modules 04 and 05.

---

*Reference patterns. Public-source, personal-capacity, advisory and organizational only. Queries must be adapted to your own schema and tested on your own warehouse before use. Outputs are review candidates for human adjudication, never payment or denial dispositions. Confirm every regulatory anchor against its primary source.*