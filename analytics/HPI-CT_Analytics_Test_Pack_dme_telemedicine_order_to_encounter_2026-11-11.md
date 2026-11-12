# Analytics Test Pack - DMEPOS and telemedicine - the order-to-encounter gap

*who ordered the item, did they ever meet the patient, and was it delivered*

*Generated 2026-11-11 from the HPI-CT control library (`data/controls.csv`, 173 controls). 16 detection tests, paired to 16 primary controls. Reference patterns, not programs.*

## Before you run anything

**These queries produce review candidates, never payment or denial decisions.** Every hit is a question for a human.

**The toolkit never receives your data.** These are patterns you run inside your own environment. No claim data, no PHI, no identifiers leave your systems.

**Every query is a pattern, not a program.** Field names are generic; yours will differ. Expect to rewrite the FROM and JOIN clauses and to remap service categories to your own taxonomy.

**Every threshold is a starting point, not a standard.** Where a threshold is arithmetic it is labelled as such. Everywhere else, measure your own distribution first.

Read `PREFACE_how_to_run_these.md` before the first run.

## Scope of this family

Durable medical equipment, prosthetics, orthotics and supplies, and the telemedicine encounters
increasingly used to originate the orders behind them. The controlling question in this family
is narrow and answerable: an item was billed, so who ordered it, what relationship did that
person have with the beneficiary, was the order written before delivery, was prior
authorisation obtained where required, and did the item actually arrive. This is the pattern
behind Operation Gold Rush and the telefraud prosecutions - orders generated at industrial
scale by clinicians with no treating relationship, signed after a scripted telephone contact,
fanned out across supplier networks.

This is the family with the tightest fit between control library and analytics. Module 04
carries DMEPOS-specific controls that name the exact artifacts a query can look for - a face-
to-face note, a written order prior to delivery, a unique tracking number on the claim, a
proof-of-delivery record. Almost nothing here is a statistical inference; most tests ask
whether a required artifact exists and whether the dates on it are in the right order. That
makes it the most immediately runnable family in the pack, and the one where a hit is hardest
to argue with.

## Tests at a glance

| # | Test | Primary control | Also touches |
|---|---|---|---|
| 1 | Order-to-encounter gap - DMEPOS with no encounter between beneficiary and ordering provider | **PA-04** | CD-08, PI-20 |
| 2 | Ordering-provider fan-out across unrelated suppliers | **PI-20** | PE-15, PI-03 |
| 3 | Telehealth contact immediately preceding an order, with no other relationship | **CD-18** | PA-04, CD-03 |
| 4 | Delivery before the written order - WOPD sequence failure | **CD-08** | PA-04, CD-09 |
| 5 | Prior authorisation affirmed but no UTN linked to the claim | **PA-05** | PA-01, PA-18 |
| 6 | Item on the Required PA List furnished with no prior authorisation at all | **PA-01** | PA-02, PA-05 |
| 7 | Required PA List currency - newly listed items still paying without authorisation | **PA-02** | PA-01, CB-17 |
| 8 | Proof of delivery absent or inconsistent with the affirmation | **PA-18** | CD-09, PA-05 |
| 9 | Rental continuation and repairs without current documented medical need | **PA-07** | CD-10, CD-03 |
| 10 | Capped-rental cap exceeded - rental payments beyond the purchase transition | **CB-08** | CB-07, PA-07 |
| 11 | Resupply before the item's reasonable useful lifetime | **CD-10** | CB-08, PA-07 |
| 12 | High-denial item categories with no documentation checklist in use | **CD-05** | PA-08, CD-15 |
| 13 | Medical-necessity rationale absent at the coverage determination | **PA-06** | PA-03, CD-03 |
| 14 | DMEPOS supplier standards - surety bond and accreditation currency | **PE-17** | PE-14, PE-02 |
| 15 | ABN not issued where non-affirmation or denial was likely | **PA-13** | PA-06, PA-12 |
| 16 | Items delivered during an inpatient or institutional stay | **CD-17** | CD-09, PA-18 |

> **Control-ID validation:** every control referenced above resolves to a real ID in the current control library. See the gap notes at the end for controls this family *needs* but the library does not yet have.

---

## 1. Order-to-encounter gap - DMEPOS with no encounter between beneficiary and ordering provider

| | |
|---|---|
| **Paired control** | **PA-04** - Face-to-face encounter & Written Order Prior to Delivery (WOPD) verification for applicable DMEPOS |
| **Control type** | Preventive |
| **Module** | Module 04 - Prior Authorization |
| **Owner** | PA coordinator |
| **Frequency** | Per order |
| **System touchpoint** | EHR / PA portal |
| **Evidence artifact** | F2F note + WOPD on file |
| **Control failure mode** | Delivery before a valid written order |
| **Regulatory anchor** | 42 CFR 410.38 [binding] |
| **Secondary controls** | CD-08 - Valid-order verification: a treating physician/practitioner order is …; PI-20 - Provider NPI/credential-footprint consistency monitoring - flag … |

**1 · Data hypothesis.** Applicable DMEPOS items require a documented face-to-face encounter with the ordering
practitioner. Where face-to-face and written-order verification is absent, items are ordered by
clinicians who never saw the beneficiary - the defining signature of the telemarketing-driven
DME schemes. The pattern is visible without any documentation at all: no claim anywhere shows
the ordering provider and the beneficiary in the same encounter.

**2 · What to check.** For each paid DMEPOS claim, look back over the applicable window for any professional or
telehealth encounter between the beneficiary and the ordering NPI. Report items with no
encounter at all, and separately those where the only encounter is a brief telehealth contact
on or immediately before the order date.

**3 · Reference SQL.**

```sql
-- TEST 1 - DMEPOS with no encounter between beneficiary and ordering provider
-- Applies the F2F window, separates 'no encounter at all' from 'telehealth-only
-- immediately before the order', and carries the supplier so the roll-up works in both
-- directions.
-- 42 CFR 410.38. Params: :period_start, :period_end, :f2f_window_days (180)

WITH dme AS (
  SELECT
      d.claim_id, d.claim_line_id, d.beneficiary_id,
      d.ordering_npi, d.billing_npi AS supplier_npi,
      d.hcpcs, d.service_date, d.paid_amount
  FROM claim_lines d
  WHERE d.service_date BETWEEN :period_start AND :period_end
    AND d.adjudication_status = 'PAID'
    AND d.claim_type = 'DMEPOS'
    AND d.ordering_npi IS NOT NULL
)
SELECT
    dme.*,
    enc.any_encounters,
    enc.inperson_encounters,
    enc.telehealth_encounters,
    enc.last_encounter_date,
    CASE
      WHEN COALESCE(enc.any_encounters, 0) = 0            THEN 'NO_ENCOUNTER_AT_ALL'
      WHEN COALESCE(enc.inperson_encounters, 0) = 0
       AND enc.telehealth_encounters > 0                  THEN 'TELEHEALTH_ONLY'
      ELSE 'ENCOUNTER_PRESENT'
    END AS encounter_class
FROM dme
LEFT JOIN LATERAL (
    SELECT
        COUNT(*) AS any_encounters,
        SUM(CASE WHEN e.place_of_service NOT IN ('02','10') THEN 1 ELSE 0 END) AS inperson_encounters,
        SUM(CASE WHEN e.place_of_service IN ('02','10')     THEN 1 ELSE 0 END) AS telehealth_encounters,
        MAX(e.service_date) AS last_encounter_date
    FROM claim_lines e
    WHERE e.beneficiary_id = dme.beneficiary_id
      AND e.rendering_npi  = dme.ordering_npi
      AND e.claim_type IN ('PROFESSIONAL','TELEHEALTH')
      AND e.cpt BETWEEN '99202' AND '99215'
      AND e.adjudication_status = 'PAID'
      AND e.service_date BETWEEN dme.service_date - :f2f_window_days AND dme.service_date
) enc ON TRUE
WHERE COALESCE(enc.any_encounters, 0) = 0
   OR COALESCE(enc.inperson_encounters, 0) = 0
ORDER BY encounter_class, dme.paid_amount DESC;

-- Roll up BOTH ways: by ordering_npi (test 2) and by supplier_npi. A supplier whose
-- entire book has no encounter footprint is a different proposition from one with a
-- handful of gaps, and the supplier roll-up is the one that sizes the exposure.
```

**4 · Threshold guidance.** :f2f_window_days = 180 is a generous starting window; the applicable requirement is six months
for many items but read it off the current rule for the item category rather than adopting a
single value. Two structural caveats. Claims-based inference understates the encounter
footprint: care delivered under another payer, in a facility bundled payment, or by a covered
non-billing encounter leaves no trace, so NO_ENCOUNTER_AT_ALL is not proof of no encounter. And
TELEHEALTH_ONLY is not itself improper - telehealth satisfies the face-to-face requirement for
many items - which is why it is separated rather than flagged: it is the population test 3
examines more closely, not a finding on its own. Tune to your population; a starting point, not
a standard.

**5 · Paired control + failure mode.** PA-04 (Face-to-face encounter & Written Order Prior to Delivery verification for applicable DMEPOS). Secondary: CD-08 (Valid-order verification), PI-20 (Provider NPI/credential-footprint consistency monitoring).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No face-to-face or WOPD verification operates. PA-04's F2F note + WOPD on file artifact does not exist. |
| **Designed-not-operating** | PA-04 is per-order; a population of delivered items has no corresponding verification record. |
| **Operated-but-bypassed** | Items were delivered on a supplier-prepared order form signed after the fact, with the face-to-face requirement treated as a formality. |
| **Operated-but-ineffective** | Verification confirms a face-to-face note exists in the file without testing whether the signing clinician had any relationship with the beneficiary - which is precisely what a generated order is built to satisfy. |
| **Evidence-missing** | Encounters occurred but no retained F2F documentation ties them to the order. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 2. Ordering-provider fan-out across unrelated suppliers

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
| **Secondary controls** | PE-15 - New-provider billing-velocity watch (first 60-90 days); PI-03 - Multi-source link analysis (claims+enrollment+encounter+external) |

**1 · Data hypothesis.** A clinician selling signatures produces a signature pattern no clinical practice produces: very
high order volume, spread across many unrelated suppliers, for beneficiaries concentrated in
geographies the clinician has no presence in. Where footprint monitoring is absent, each
individual order looks ordinary and only the aggregate is diagnostic. This is the Gold Rush
structure.

**2 · What to check.** For each ordering provider on DMEPOS claims, count orders, distinct beneficiaries, distinct
suppliers and distinct beneficiary states, and compute the share of orders with no encounter
footprint. Rank by supplier fan-out and geographic spread rather than by volume alone.

**3 · Reference SQL.**

```sql
-- TEST 2 - Ordering-provider fan-out and geographic spread on DMEPOS
-- Params: :period_start, :period_end, :min_orders (50), :supplier_fanout_cut (5), :f2f_window_days (180)

WITH orders AS (
  SELECT
      d.ordering_npi, d.billing_npi AS supplier_npi, d.beneficiary_id,
      d.claim_line_id, d.service_date, d.paid_amount,
      b.state_code AS beneficiary_state,
      CASE WHEN EXISTS (
        SELECT 1 FROM claim_lines e
        WHERE e.beneficiary_id = d.beneficiary_id
          AND e.rendering_npi  = d.ordering_npi
          AND e.claim_type IN ('PROFESSIONAL','TELEHEALTH')
          AND e.adjudication_status = 'PAID'
          AND e.service_date BETWEEN d.service_date - :f2f_window_days AND d.service_date
      ) THEN 1 ELSE 0 END AS has_encounter
  FROM claim_lines d
  JOIN beneficiary b ON b.beneficiary_id = d.beneficiary_id
  WHERE d.service_date BETWEEN :period_start AND :period_end
    AND d.adjudication_status = 'PAID'
    AND d.claim_type = 'DMEPOS'
    AND d.ordering_npi IS NOT NULL
)
SELECT
    o.ordering_npi, pr.specialty, pr.practice_state,
    COUNT(*)                                   AS orders,
    COUNT(DISTINCT o.beneficiary_id)           AS beneficiaries,
    COUNT(DISTINCT o.supplier_npi)             AS distinct_suppliers,
    COUNT(DISTINCT o.beneficiary_state)        AS distinct_beneficiary_states,
    SUM(CASE WHEN o.beneficiary_state <> pr.practice_state THEN 1 ELSE 0 END) * 1.0
      / NULLIF(COUNT(*), 0)                    AS out_of_state_share,
    ROUND(1 - AVG(o.has_encounter), 3)         AS no_encounter_share,
    SUM(o.paid_amount)                         AS ordered_paid,
    ROUND(COUNT(*) * 1.0 / NULLIF(COUNT(DISTINCT o.service_date), 0), 1) AS orders_per_active_day
FROM orders o
LEFT JOIN provider pr ON pr.npi = o.ordering_npi
GROUP BY o.ordering_npi, pr.specialty, pr.practice_state
HAVING COUNT(*) >= :min_orders
   AND COUNT(DISTINCT o.supplier_npi) >= :supplier_fanout_cut
ORDER BY no_encounter_share DESC, ordered_paid DESC;

-- The conjunction is the signal, not any single column: high supplier fan-out AND high
-- out-of-state share AND high no-encounter share AND high orders-per-active-day. Any one
-- alone has ordinary explanations; all four together do not.
```

**4 · Threshold guidance.** :min_orders = 50 and :supplier_fanout_cut = 5 distinct suppliers are starting cuts. Read the
four signal columns together rather than thresholding any one: a hospitalist orders for many
suppliers, a telehealth practice legitimately serves many states, and a busy practice has high
orders-per-day. What no ordinary practice produces is all four at once alongside a no-encounter
share near 1.0. `orders_per_active_day` is the most under-appreciated column - a clinician
signing hundreds of orders on a handful of days is describing a batch-signing workflow, not a
clinical one. Tune to your population; a starting point, not a standard.

**5 · Paired control + failure mode.** PI-20 (Provider NPI/credential-footprint consistency monitoring). Secondary: PE-15 (New-provider billing-velocity watch), PI-03 (Multi-source link analysis).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No ordering-provider footprint monitoring. PI-20's alert log does not cover the ordering role. |
| **Designed-not-operating** | PI-20 is continuous; no alerts covering DMEPOS ordering were produced. |
| **Operated-but-bypassed** | Fan-out was identified and no review, education or ordering-privilege action followed. |
| **Operated-but-ineffective** | The monitor evaluates billing and rendering NPIs only and never the ordering NPI, so the party generating the demand is outside its scope entirely. |
| **Evidence-missing** | Alerts fire with no retained disposition. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 3. Telehealth contact immediately preceding an order, with no other relationship

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
| **Secondary controls** | PA-04 - Face-to-face encounter & Written Order Prior to Delivery (WOPD) …; CD-03 - Reasonable-and-necessary documentation control: the medical record … |

**1 · Data hypothesis.** Telehealth satisfies the face-to-face requirement for many items, which is exactly why order-
generation schemes route through it. Where the service-modality documentation control is
absent, the pattern is a single brief telehealth contact on or within a day or two of the
order, from a clinician with no prior or subsequent relationship with the beneficiary, followed
immediately by a high-value item.

**2 · What to check.** Identify DMEPOS orders whose only qualifying encounter is a telehealth contact within a short
window before the order, and test whether that clinician has any other relationship with the
beneficiary before or after. Report the encounter's billed level as a proxy for its substance.

**3 · Reference SQL.**

```sql
-- TEST 3 - Telehealth-originated orders with no surrounding relationship
-- Params: :period_start, :period_end, :proximity_days (7), :relationship_window_days (365)

WITH tele_originated AS (
  SELECT
      d.claim_line_id, d.beneficiary_id, d.ordering_npi, d.billing_npi AS supplier_npi,
      d.hcpcs, d.service_date AS order_service_date, d.paid_amount,
      t.claim_line_id AS telehealth_line_id,
      t.service_date  AS telehealth_date,
      t.cpt           AS telehealth_cpt,
      t.paid_amount   AS telehealth_paid,
      (d.service_date - t.service_date) AS days_order_after_contact
  FROM claim_lines d
  JOIN claim_lines t
    ON  t.beneficiary_id = d.beneficiary_id
    AND t.rendering_npi  = d.ordering_npi
    AND t.place_of_service IN ('02','10')
    AND t.adjudication_status = 'PAID'
    AND t.service_date BETWEEN d.service_date - :proximity_days AND d.service_date
  WHERE d.service_date BETWEEN :period_start AND :period_end
    AND d.adjudication_status = 'PAID'
    AND d.claim_type = 'DMEPOS'
)
SELECT
    tо.*,
    -- any other contact with this clinician, before or after, of any modality
    (SELECT COUNT(*) FROM claim_lines o
      WHERE o.beneficiary_id  = tо.beneficiary_id
        AND o.rendering_npi   = tо.ordering_npi
        AND o.claim_line_id  <> tо.telehealth_line_id
        AND o.adjudication_status = 'PAID'
        AND o.service_date BETWEEN tо.order_service_date - :relationship_window_days
                               AND tо.order_service_date + :relationship_window_days)
      AS other_contacts_with_clinician,
    CASE WHEN tо.telehealth_cpt IN ('99202','99211','99212') THEN 'LOW_LEVEL_CONTACT'
         ELSE 'HIGHER_LEVEL_CONTACT' END AS contact_substance
FROM tele_originated tо
WHERE NOT EXISTS (
    SELECT 1 FROM claim_lines o
    WHERE o.beneficiary_id  = tо.beneficiary_id
      AND o.rendering_npi   = tо.ordering_npi
      AND o.claim_line_id  <> tо.telehealth_line_id
      AND o.adjudication_status = 'PAID'
      AND o.service_date BETWEEN tо.order_service_date - :relationship_window_days
                             AND tо.order_service_date + :relationship_window_days)
ORDER BY tо.paid_amount DESC;
```

**4 · Threshold guidance.** :proximity_days = 7 and :relationship_window_days = 365. Be careful with the framing here,
because the pattern this test describes is also what legitimate specialist telehealth
consultation looks like: a patient is referred, seen once remotely, an appropriate item is
ordered, and the relationship ends. That is good care, not fraud. The discriminating factors
are volume at the *clinician* level (test 2), the value of the item relative to the substance
of the contact, and whether the beneficiary was solicited - which claims data cannot show at
all. Treat individual rows as context for a clinician-level review, never as findings in
themselves. Tune to your population; a starting point, not a standard.

**5 · Paired control + failure mode.** CD-18 (Service-modality documentation control - note and claim modifier support the billed modality). Secondary: PA-04 (Face-to-face encounter & WOPD verification), CD-03 (Reasonable-and-necessary documentation).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No modality documentation control operates. CD-18's modality-supporting note + modifier artifact does not exist. |
| **Designed-not-operating** | CD-18 is per-service; a population of telehealth encounters behind DMEPOS orders has no modality documentation. |
| **Operated-but-bypassed** | Modality was recorded correctly and no one asked whether the encounter supported the order that followed it. |
| **Operated-but-ineffective** | The control tests that the modifier matches the place of service, which a scripted telehealth contact satisfies perfectly - it says nothing about clinical substance. |
| **Evidence-missing** | No retained note evidencing what occurred in the encounter. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 4. Delivery before the written order - WOPD sequence failure

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
| **Secondary controls** | PA-04 - Face-to-face encounter & Written Order Prior to Delivery (WOPD) …; CD-09 - Proof-of-delivery / proof-of-service capture and retention with all … |

**1 · Data hypothesis.** For applicable items the written order must precede delivery. Where valid-order verification is
absent or checks only that an order exists, items are delivered first and the order is papered
afterward - a sequence failure that is invisible to any control that looks only for the
presence of a document.

**2 · What to check.** Compare the order signature date to the delivery date for every item subject to the written-
order-prior-to-delivery requirement, and report deliveries preceding signature, orders with no
signature date, and orders signed by a provider who was not active on the signature date.

**3 · Reference SQL.**

```sql
-- TEST 4 - Written order date versus delivery date
-- 42 CFR 410.32; 42 CFR 410.38. Params: :period_start, :period_end
-- Needs provider.enrollment_end_date (or an enrollment history): provider status is judged on the
-- signature date, not today.

SELECT
    d.claim_line_id, d.beneficiary_id, d.billing_npi AS supplier_npi,
    d.ordering_npi, d.hcpcs, d.paid_amount,
    o.order_id, o.signature_date, o.order_received_date,
    pod.delivery_date,
    (pod.delivery_date - o.signature_date) AS days_signature_to_delivery,
    p.enrollment_status AS ordering_provider_status,
    CASE
      WHEN o.order_id IS NULL                        THEN 'NO_ORDER_RECORD'
      WHEN o.signature_date IS NULL                  THEN 'ORDER_UNSIGNED'
      WHEN pod.delivery_date IS NULL                 THEN 'NO_DELIVERY_RECORD'
      WHEN pod.delivery_date < o.signature_date      THEN 'DELIVERED_BEFORE_ORDER_SIGNED'
      -- death first: the cleanest signal, and it must not be shadowed by the not-active label
      WHEN p.date_of_death IS NOT NULL
           AND o.signature_date > p.date_of_death    THEN 'SIGNED_AFTER_PROVIDER_DEATH'
      -- status on the signature date, not today's status: use the enrollment end date when held
      WHEN (p.enrollment_end_date IS NOT NULL AND o.signature_date > p.enrollment_end_date)
        OR (p.enrollment_end_date IS NULL AND p.enrollment_status <> 'ACTIVE')
                                                     THEN 'ORDERING_PROVIDER_NOT_ACTIVE'
    END AS wopd_defect
FROM claim_lines d
JOIN wopd_required_item w ON w.hcpcs = d.hcpcs
                         AND d.service_date BETWEEN w.effective_date
                                                AND COALESCE(w.termination_date, DATE '9999-12-31')
LEFT JOIN dme_order o        ON o.claim_id = d.claim_id
LEFT JOIN proof_of_delivery pod ON pod.claim_id = d.claim_id
LEFT JOIN provider p         ON p.npi = d.ordering_npi
WHERE d.service_date BETWEEN :period_start AND :period_end
  AND d.adjudication_status = 'PAID'
  AND d.claim_type = 'DMEPOS'
  AND (o.order_id IS NULL
       OR o.signature_date IS NULL
       OR pod.delivery_date IS NULL
       OR pod.delivery_date < o.signature_date
       OR (p.enrollment_end_date IS NOT NULL AND o.signature_date > p.enrollment_end_date)
       OR (p.enrollment_end_date IS NULL AND p.enrollment_status <> 'ACTIVE')
       OR (p.date_of_death IS NOT NULL AND o.signature_date > p.date_of_death))
ORDER BY wopd_defect, d.paid_amount DESC;
```

**4 · Threshold guidance.** Categorical throughout - dates are either in sequence or they are not. Two practical notes.
NO_ORDER_RECORD and NO_DELIVERY_RECORD will usually reflect documents held by the supplier and
not surfaced in the payer's systems, so establish your document-capture completeness before
treating them as findings; they are more often an evidence-availability question than a
compliance one. SIGNED_AFTER_PROVIDER_DEATH is low-volume and near-conclusive, and should be
worked first regardless of dollar value - it is the single cleanest signal in this family. Tune
to your population; a starting point, not a standard.

**5 · Paired control + failure mode.** CD-08 (Valid-order verification - signed order on file before the service, source-verified ordering-provider identity). Secondary: PA-04 (Face-to-face & WOPD verification), CD-09 (Proof-of-delivery capture and retention).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No order verification before delivery. CD-08's signed-order artifact does not exist for this category. |
| **Designed-not-operating** | CD-08 is per-service; a population of delivered items has no verification record. |
| **Operated-but-bypassed** | Delivery proceeded on a verbal or pending order and the written order was obtained afterward. |
| **Operated-but-ineffective** | The check confirms an order document exists without comparing its signature date to the delivery date - presence, not sequence, which is the whole requirement. |
| **Evidence-missing** | Orders exist at the supplier but are not retrievable within the retention requirement. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 5. Prior authorisation affirmed but no UTN linked to the claim

| | |
|---|---|
| **Paired control** | **PA-05** - Affirmation/UTN capture & claim linkage: attach Unique Tracking Number to the claim before billing |
| **Control type** | Detective |
| **Module** | Module 04 - Prior Authorization |
| **Owner** | Billing |
| **Frequency** | Per claim |
| **System touchpoint** | Claims + PA system |
| **Evidence artifact** | UTN present on claim |
| **Control failure mode** | Billing without an affirmed PA |
| **Regulatory anchor** | CMS PIM 100-08 (Program Integrity Manual) [guidance] |
| **Secondary controls** | PA-01 - Prospective PA-scoping check: match each order against current DMEPOS …; PA-18 - Proof-of-delivery / service-vs-affirmation post-payment match |

**1 · Data hypothesis.** Where the unique tracking number is not captured and linked, claims bill without an affirmed
prior authorisation - or, just as importantly, with a UTN belonging to a different item,
beneficiary or date. PA-05's stated failure mode is 'billing without an affirmed PA', and the
linkage is the control.

**2 · What to check.** For claims in PA-required categories, test that a UTN is present, that it resolves to a real
affirmation, and that the affirmation matches on beneficiary, HCPCS and date range. Report each
mismatch class separately.

**3 · Reference SQL.**

```sql
-- TEST 5 - UTN presence and affirmation linkage integrity
-- Params: :period_start, :period_end

SELECT
    d.claim_line_id, d.beneficiary_id, d.billing_npi AS supplier_npi,
    d.hcpcs, d.service_date, d.paid_amount,
    d.utn,
    a.affirmation_id, a.decision, a.decision_date,
    a.beneficiary_id AS affirmed_beneficiary, a.hcpcs AS affirmed_hcpcs,
    a.valid_from, a.valid_to,
    CASE
      WHEN d.utn IS NULL                             THEN 'NO_UTN_ON_CLAIM'
      WHEN a.affirmation_id IS NULL                  THEN 'UTN_DOES_NOT_RESOLVE'
      WHEN a.decision <> 'AFFIRMED'                  THEN 'UTN_RESOLVES_TO_NON_AFFIRMATION'
      WHEN a.beneficiary_id <> d.beneficiary_id      THEN 'UTN_BENEFICIARY_MISMATCH'
      WHEN a.hcpcs <> d.hcpcs                        THEN 'UTN_ITEM_MISMATCH'
      WHEN d.service_date NOT BETWEEN a.valid_from AND a.valid_to
                                                     THEN 'UTN_OUTSIDE_VALIDITY_WINDOW'
    END AS utn_defect
FROM claim_lines d
JOIN pa_required_item pr
  ON  pr.hcpcs = d.hcpcs
  AND d.service_date BETWEEN pr.effective_date
                         AND COALESCE(pr.termination_date, DATE '9999-12-31')
LEFT JOIN pa_affirmation a ON a.utn = d.utn
WHERE d.service_date BETWEEN :period_start AND :period_end
  AND d.adjudication_status = 'PAID'
  AND (d.utn IS NULL
       OR a.affirmation_id IS NULL
       OR a.decision <> 'AFFIRMED'
       OR a.beneficiary_id <> d.beneficiary_id
       OR a.hcpcs <> d.hcpcs
       OR d.service_date NOT BETWEEN a.valid_from AND a.valid_to)
ORDER BY utn_defect, d.paid_amount DESC;
```

**4 · Threshold guidance.** Categorical against the PA-required item list, and that list's currency is the dependency - see
test 7. The mismatch classes are worth keeping separate because they mean different things:
NO_UTN_ON_CLAIM is usually a billing-configuration failure, UTN_BENEFICIARY_MISMATCH and
UTN_ITEM_MISMATCH suggest a UTN reused across claims, and UTN_RESOLVES_TO_NON_AFFIRMATION means
a non-affirmed request was billed anyway. The reuse classes are small in volume and high in
significance. Tune to your population; a starting point, not a standard.

**5 · Paired control + failure mode.** PA-05 (Affirmation/UTN capture & claim linkage). Secondary: PA-01 (Prospective PA-scoping check), PA-18 (Proof-of-delivery / service-vs-affirmation post-payment match).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No UTN capture or linkage. PA-05's 'UTN present on claim' evidence does not exist. |
| **Designed-not-operating** | The linkage is specified per claim; a population of PA-required claims carries no UTN. |
| **Operated-but-bypassed** | Claims without a UTN were released to billing to avoid a hold. |
| **Operated-but-ineffective** | The control checks that a UTN field is populated but never validates it against the affirmation record, so a reused or fabricated number passes. |
| **Evidence-missing** | Affirmations were obtained but no retained linkage evidences which claim they belong to. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 6. Item on the Required PA List furnished with no prior authorisation at all

| | |
|---|---|
| **Paired control** | **PA-01** - Prospective PA-scoping check: match each order against current DMEPOS Required PA List & OPD PA list before delivery/service |
| **Control type** | Preventive |
| **Module** | Module 04 - Prior Authorization |
| **Owner** | Intake / PA coordinator |
| **Frequency** | Per order |
| **System touchpoint** | PA workflow + item master |
| **Evidence artifact** | PA-applicability determination |
| **Control failure mode** | Furnishing a listed item/service without required PA (auto-denial) |
| **Regulatory anchor** | 42 CFR 414.234 [binding]; 42 CFR 410.38 [binding] |
| **Secondary controls** | PA-02 - Master List / Required PA List update governance: monitor annual …; PA-05 - Affirmation/UTN capture & claim linkage: attach Unique Tracking … |

**1 · Data hypothesis.** The prospective scoping check exists to catch, before delivery, that an item requires prior
authorisation. Where it is absent, items are furnished and billed with no PA request ever
submitted - not a mismatched UTN but no request in the system at all. PA-01's failure mode
names the consequence: furnishing a listed item without required PA, which auto-denies.

**2 · What to check.** For each paid claim in a PA-required category, test whether any PA request exists for that
beneficiary and item in the applicable window, regardless of outcome. Report by supplier, since
a scoping failure is an operational pattern.

**3 · Reference SQL.**

```sql
-- TEST 6 - PA-required items with no PA request of any kind
-- 42 CFR 414.234. Params: :period_start, :period_end, :request_window_days (180)

WITH pa_required_claims AS (
  SELECT d.claim_line_id, d.claim_id, d.beneficiary_id, d.billing_npi AS supplier_npi,
         d.hcpcs, d.service_date, d.paid_amount, pr.list_name, pr.added_to_list_date
  FROM claim_lines d
  JOIN pa_required_item pr
    ON  pr.hcpcs = d.hcpcs
    AND d.service_date BETWEEN pr.effective_date
                           AND COALESCE(pr.termination_date, DATE '9999-12-31')
  WHERE d.service_date BETWEEN :period_start AND :period_end
    AND d.adjudication_status = 'PAID'
)
SELECT
    c.supplier_npi,
    COUNT(*)                          AS claims_without_any_pa,
    COUNT(DISTINCT c.hcpcs)           AS distinct_items,
    COUNT(DISTINCT c.beneficiary_id)  AS beneficiaries,
    SUM(c.paid_amount)                AS paid_amount,
    MIN(c.service_date)               AS earliest,
    MAX(c.service_date)               AS latest,
    -- how much of the supplier's PA-required volume this represents
    ROUND(COUNT(*) * 1.0 / NULLIF(
      (SELECT COUNT(*) FROM pa_required_claims x WHERE x.supplier_npi = c.supplier_npi), 0), 3)
      AS share_of_pa_required_volume
FROM pa_required_claims c
WHERE NOT EXISTS (
    SELECT 1 FROM pa_request r
    WHERE r.beneficiary_id = c.beneficiary_id
      AND r.hcpcs          = c.hcpcs
      AND r.request_date BETWEEN c.service_date - :request_window_days AND c.service_date
)
GROUP BY c.supplier_npi
ORDER BY paid_amount DESC;
```

**4 · Threshold guidance.** Categorical. The `share_of_pa_required_volume` column is what distinguishes an operational gap
from an isolated miss - a supplier at 2% has a process with holes, one at 60% does not have a
process. Two exclusions to build first: emergency and expedited furnishing pathways that permit
delivery before authorisation, and items recently added to the list where a transition period
applies (test 7 covers the list-currency side of that). Without them the newest list additions
will dominate and misattribute a governance lag as a supplier failure. Tune to your population;
a starting point, not a standard.

**5 · Paired control + failure mode.** PA-01 (Prospective PA-scoping check against the current DMEPOS Required PA List and OPD PA list before delivery). Secondary: PA-02 (Required PA List update governance), PA-05 (Affirmation/UTN capture & claim linkage).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No scoping check before delivery. PA-01's PA-applicability determination does not exist. |
| **Designed-not-operating** | PA-01 is per-order; a population of orders for listed items has no applicability determination. |
| **Operated-but-bypassed** | The item was identified as PA-required and furnished anyway, ahead of a decision. |
| **Operated-but-ineffective** | The check runs against a stale item master, so newly listed items are never identified as requiring PA - the failure belongs upstream to PA-02. |
| **Evidence-missing** | Determinations were made but no retained record shows which orders were scoped. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 7. Required PA List currency - newly listed items still paying without authorisation

| | |
|---|---|
| **Paired control** | **PA-02** - Master List / Required PA List update governance: monitor annual Federal Register + CMS PA site; refresh item-to-HCPCS mapping |
| **Control type** | Detective |
| **Module** | Module 04 - Prior Authorization |
| **Owner** | Compliance + coding |
| **Frequency** | Quarterly |
| **System touchpoint** | Item master / mapping table |
| **Evidence artifact** | List-version change log |
| **Control failure mode** | Stale PA list misses newly-listed items |
| **Regulatory anchor** | 42 CFR 414.234 [binding]; Federal Register 2026-00487 (Jan 13, 2026) [binding] |
| **Secondary controls** | PA-01 - Prospective PA-scoping check: match each order against current DMEPOS …; CB-17 - Prepayment edit maintenance & tuning from emerging vulnerabilities |

**1 · Data hypothesis.** The Required PA List and Master List change through the Federal Register and the CMS PA site.
Where update governance is absent, the item master goes stale and newly listed items continue
to pay unauthorised. This is the upstream cause of most of test 6's volume, and distinguishing
the two is the point of running both.

**2 · What to check.** For each item added to the list during or before the period, measure the interval between the
list effective date and the date your item master reflected it, and quantify the claims paid
without PA inside that lag.

**3 · Reference SQL.**

```sql
-- TEST 7 - Item-master lag behind the published Required PA List
-- Params: :period_start, :period_end

WITH list_changes AS (
  SELECT pl.hcpcs, pl.list_name, pl.published_date, pl.effective_date,
         im.item_master_updated_date
  FROM published_pa_list pl
  LEFT JOIN item_master_pa_flag im
         ON im.hcpcs = pl.hcpcs AND im.list_name = pl.list_name
  WHERE pl.effective_date BETWEEN :period_start - 730 AND :period_end
)
SELECT
    lc.hcpcs, lc.list_name, lc.published_date, lc.effective_date,
    lc.item_master_updated_date,
    CASE WHEN lc.item_master_updated_date IS NULL THEN NULL
         ELSE lc.item_master_updated_date - lc.effective_date END AS update_lag_days,
    CASE
      WHEN lc.item_master_updated_date IS NULL                    THEN 'NEVER_UPDATED'
      WHEN lc.item_master_updated_date > lc.effective_date        THEN 'UPDATED_LATE'
      ELSE 'TIMELY'
    END AS governance_status,
    -- exposure inside the lag: paid claims for the item with no PA request
    (SELECT COALESCE(SUM(cl.paid_amount), 0)
       FROM claim_lines cl
      WHERE cl.hcpcs = lc.hcpcs
        AND cl.adjudication_status = 'PAID'
        AND cl.service_date >= lc.effective_date
        AND cl.service_date <  COALESCE(lc.item_master_updated_date, CURRENT_DATE)
        AND NOT EXISTS (SELECT 1 FROM pa_request r
                        WHERE r.beneficiary_id = cl.beneficiary_id
                          AND r.hcpcs = cl.hcpcs)) AS paid_unauthorised_in_lag
FROM list_changes lc
WHERE lc.item_master_updated_date IS NULL
   OR lc.item_master_updated_date > lc.effective_date
ORDER BY paid_unauthorised_in_lag DESC;
```

**4 · Threshold guidance.** No threshold - the lag is a measured number and the exposure inside it is a measured dollar
amount. This output is a governance finding by construction and belongs to compliance and
coding, not to any supplier's review queue. Note the published date and effective date are
different things and both matter: a list published with thirty days' notice and updated on day
thirty-one is a different control posture from one published with six months' notice and
updated late anyway. Read `published_date` alongside the lag before characterising the failure.
Tune to your population; a starting point, not a standard.

**5 · Paired control + failure mode.** PA-02 (Master List / Required PA List update governance - monitor annual Federal Register + CMS PA site, refresh item-to-HCPCS mapping). Secondary: PA-01 (Prospective PA-scoping check), CB-17 (Prepayment edit maintenance & tuning).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No list monitoring. PA-02's list-version change log does not exist. |
| **Designed-not-operating** | PA-02 is quarterly; the change log shows no entry against a published list update. |
| **Operated-but-bypassed** | The update was identified and the item-master change was deferred. |
| **Operated-but-ineffective** | Monitoring covers the annual Federal Register notice but not interim CMS PA-site updates, so mid-year additions are missed. |
| **Evidence-missing** | Updates occurred but no dated change log records what changed and when. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 8. Proof of delivery absent or inconsistent with the affirmation

| | |
|---|---|
| **Paired control** | **PA-18** - Proof-of-delivery / service-vs-affirmation post-payment match |
| **Control type** | Detective |
| **Module** | Module 04 - Prior Authorization |
| **Owner** | PI analyst |
| **Frequency** | Monthly |
| **System touchpoint** | Claims + POD records |
| **Evidence artifact** | POD-to-PA reconciliation |
| **Control failure mode** | Billing for items not delivered / not matching the PA |
| **Regulatory anchor** | CMS PIM 100-08 (Program Integrity Manual) [guidance] |
| **Secondary controls** | CD-09 - Proof-of-delivery / proof-of-service capture and retention with all …; PA-05 - Affirmation/UTN capture & claim linkage: attach Unique Tracking … |

**1 · Data hypothesis.** An affirmed prior authorisation says an item may be furnished; proof of delivery says it was.
Where the post-payment match between the two is absent, claims pay for items never delivered,
or for items different from those authorised. PA-18 exists specifically to reconcile these two
records and its failure mode names both cases.

**2 · What to check.** Reconcile paid DMEPOS claims to proof-of-delivery records and to the affirmation, testing
existence, completeness of required POD elements, and agreement on item, beneficiary and date.

**3 · Reference SQL.**

```sql
-- TEST 8 - POD existence, completeness and agreement with the affirmation
-- Program Integrity Manual Pub 100-08 Ch.5; 42 CFR 410.38.
-- Params: :period_start, :period_end, :pod_tolerance_days (2)

SELECT
    d.claim_line_id, d.beneficiary_id, d.billing_npi AS supplier_npi,
    d.hcpcs, d.service_date, d.paid_amount, d.utn,
    pod.pod_id, pod.delivery_date, pod.delivery_method,
    pod.recipient_signature_present, pod.item_description_present,
    pod.quantity_present, pod.delivery_address_present,
    a.hcpcs AS affirmed_hcpcs, a.quantity AS affirmed_quantity,
    CASE
      WHEN pod.pod_id IS NULL                                THEN 'NO_POD'
      WHEN pod.recipient_signature_present = FALSE
        OR pod.item_description_present   = FALSE
        OR pod.quantity_present           = FALSE
        OR pod.delivery_address_present   = FALSE            THEN 'POD_INCOMPLETE'
      WHEN a.affirmation_id IS NOT NULL AND a.hcpcs <> d.hcpcs
                                                             THEN 'ITEM_DIFFERS_FROM_AFFIRMATION'
      WHEN a.affirmation_id IS NOT NULL AND d.units > a.quantity
                                                             THEN 'QUANTITY_EXCEEDS_AFFIRMATION'
      WHEN pod.delivery_date > d.service_date + :pod_tolerance_days
        OR pod.delivery_date < d.service_date - :pod_tolerance_days
                                                             THEN 'DELIVERY_DATE_MISMATCH'
    END AS pod_defect
FROM claim_lines d
LEFT JOIN proof_of_delivery pod ON pod.claim_id = d.claim_id
LEFT JOIN pa_affirmation a      ON a.utn = d.utn
WHERE d.service_date BETWEEN :period_start AND :period_end
  AND d.adjudication_status = 'PAID'
  AND d.claim_type = 'DMEPOS'
  AND (pod.pod_id IS NULL
       OR pod.recipient_signature_present = FALSE
       OR pod.item_description_present   = FALSE
       OR pod.quantity_present           = FALSE
       OR pod.delivery_address_present   = FALSE
       OR (a.affirmation_id IS NOT NULL AND (a.hcpcs <> d.hcpcs OR d.units > a.quantity))
       OR ABS(pod.delivery_date - d.service_date) > :pod_tolerance_days)
ORDER BY pod_defect, d.paid_amount DESC;
```

**4 · Threshold guidance.** :pod_tolerance_days = 2 absorbs the ordinary gap between a shipment date and the billed date of
service; shipping-based delivery legitimately spans several days, so set the tolerance by
delivery method rather than globally. The NO_POD class depends entirely on whether the payer
holds POD records at all - most do not outside an audit, in which case PA-18's monthly post-
payment sample is the mechanism and this query describes what that sample should test rather
than something you can run across the whole population. ITEM_DIFFERS_FROM_AFFIRMATION is the
highest-value class and is testable without POD data. Tune to your population; a starting
point, not a standard.

**5 · Paired control + failure mode.** PA-18 (Proof-of-delivery / service-vs-affirmation post-payment match). Secondary: CD-09 (Proof-of-delivery capture and retention with all required elements), PA-05 (Affirmation/UTN capture & claim linkage).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No post-payment POD-to-PA reconciliation. PA-18's reconciliation artifact does not exist. |
| **Designed-not-operating** | PA-18 is monthly; no sample was drawn or reconciled in the period. |
| **Operated-but-bypassed** | Mismatches were identified and no recovery or supplier action followed. |
| **Operated-but-ineffective** | The match confirms a POD document exists without testing its required elements or its agreement with the affirmation. |
| **Evidence-missing** | Reconciliation performed with no retained workpaper showing which claims were matched. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 9. Rental continuation and repairs without current documented medical need

| | |
|---|---|
| **Paired control** | **PA-07** - Continued-medical-necessity re-documentation for rentals/repairs within required timeframe (12-mo PMD) [open OIG rec] |
| **Control type** | Detective |
| **Module** | Module 04 - Prior Authorization |
| **Owner** | Supplier / treating clinician |
| **Frequency** | Per event |
| **System touchpoint** | EHR + supplier records |
| **Evidence artifact** | Timely continued-need record |
| **Control failure mode** | Repairs/rentals billed without current need (OIG PMD) |
| **Regulatory anchor** | CMS PIM 100-08 (Program Integrity Manual) [guidance]; OIG A-09-20-03016 (31 May 2022) - PMD repairs; rec 22-A-09-063.07 open, .01 to .06 closed [case evidence] |
| **Secondary controls** | CD-10 - Continued-need / recertification documentation control for ongoing …; CD-03 - Reasonable-and-necessary documentation control: the medical record … |

**1 · Data hypothesis.** Rentals continue and repairs are billed on the strength of a need documented once, sometimes
years earlier. Where continued-medical-necessity re-documentation is absent, monthly rental and
repair claims accumulate with no current record - the OIG power-mobility-device repair finding
PA-07 cites is the reference case.

**2 · What to check.** For each rental month and each repair claim, compute time since the most recent continued-need
documentation and flag those beyond the applicable interval. Quantify the paid amount inside
the lapsed window, which is the exposure figure.

**3 · Reference SQL.**

```sql
-- TEST 9 - Rental months and repairs beyond the continued-need interval
-- Params: :period_start, :period_end, :cmn_interval_days (365)

WITH rental_and_repair AS (
  SELECT d.claim_line_id, d.beneficiary_id, d.billing_npi AS supplier_npi,
         d.hcpcs, d.service_date, d.paid_amount,
         CASE WHEN d.modifier_1 IN ('RR') OR d.modifier_2 IN ('RR') THEN 'RENTAL'
              WHEN d.modifier_1 IN ('RB','RP') OR d.modifier_2 IN ('RB','RP') THEN 'REPAIR'
              ELSE 'OTHER' END AS claim_nature
  FROM claim_lines d
  WHERE d.service_date BETWEEN :period_start AND :period_end
    AND d.adjudication_status = 'PAID'
    AND d.claim_type = 'DMEPOS'
),
claim_cmn AS (
  -- the latest document ON OR BEFORE each claim's service date, so a later re-documentation
  -- does not make an earlier, validly documented month look undocumented
  SELECT r.*, ec.equipment_category,
         (SELECT MAX(c.documented_date)
            FROM continued_need_document c
           WHERE c.beneficiary_id     = r.beneficiary_id
             AND c.equipment_category = ec.equipment_category
             AND c.documented_date   <= r.service_date) AS last_documented
  FROM rental_and_repair r
  JOIN equipment_category_map ec ON ec.hcpcs = r.hcpcs
  WHERE r.claim_nature IN ('RENTAL','REPAIR')
)
SELECT
    r.claim_line_id, r.beneficiary_id, r.supplier_npi, r.hcpcs,
    r.claim_nature, r.service_date, r.paid_amount,
    r.last_documented,
    CASE WHEN r.last_documented IS NULL THEN NULL
         ELSE r.service_date - r.last_documented END AS days_since_documented,
    CASE WHEN r.last_documented IS NULL THEN 'NO_CONTINUED_NEED_RECORD'
         ELSE 'STALE_CONTINUED_NEED' END AS cmn_defect
FROM claim_cmn r
WHERE r.last_documented IS NULL
   OR r.service_date - r.last_documented > :cmn_interval_days
ORDER BY r.paid_amount DESC;

-- Roll up by supplier and by beneficiary. Exposure = SUM(paid_amount) in the lapsed
-- window, which is the figure that belongs in a report; individual months are small.
```

**4 · Threshold guidance.** :cmn_interval_days = 365 reflects the twelve-month power-mobility interval PA-07 names, but
intervals differ by equipment category and by payer policy - set it per category from your own
coverage rules rather than globally. The modifier-based identification of rentals and repairs
is approximate: modifier usage varies and some suppliers bill repairs without RB/RP, so
validate the classification against a sample before trusting the counts.
NO_CONTINUED_NEED_RECORD will usually reflect documentation held clinically and not surfaced to
the payer, which makes this an evidence-retrieval finding as much as a coverage one. Tune to
your population; a starting point, not a standard.

**5 · Paired control + failure mode.** PA-07 (Continued-medical-necessity re-documentation for rentals/repairs within the required timeframe). Secondary: CD-10 (Continued-need / recertification documentation control), CD-03 (Reasonable-and-necessary documentation).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No continued-need re-documentation operates. PA-07's timely continued-need record does not exist. |
| **Designed-not-operating** | PA-07 is per-event; rentals continued and repairs were billed with no re-documentation. |
| **Operated-but-bypassed** | Re-documentation was produced as a form renewal with no reassessment of need. |
| **Operated-but-ineffective** | Documentation is tracked in the supplier's system and never blocks or flags a claim, so a lapse carries no payment consequence. |
| **Evidence-missing** | Reassessments occurred but the record is not retained or retrievable. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 10. Capped-rental cap exceeded - rental payments beyond the purchase transition

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
| **Secondary controls** | CB-07 - MUE units-of-service edit enforcement; PA-07 - Continued-medical-necessity re-documentation for rentals/repairs … |

**1 · Data hypothesis.** Capped rental items transition to ownership after a defined number of rental months, after
which continued rental billing is not payable. Where quantity and frequency edits do not track
cumulative rental months per beneficiary per item, rental billing continues past the cap - and
because each month is individually small and individually valid, nothing fires.

**2 · What to check.** Accumulate rental months per beneficiary per equipment item across the whole history, not just
the reporting period, and flag months beyond the applicable cap. Report cumulative rental paid
against the item's purchase price as a second view.

**3 · Reference SQL.**

```sql
-- TEST 10 - Rental months beyond the capped-rental limit
-- Params: :period_start, :period_end, :default_rental_cap_months (13)

WITH rental_months AS (
  SELECT
      d.beneficiary_id, ec.equipment_category, d.hcpcs, d.billing_npi AS supplier_npi,
      DATE_TRUNC('month', d.service_date) AS rental_month,
      SUM(d.paid_amount) AS month_paid
  FROM claim_lines d
  JOIN equipment_category_map ec ON ec.hcpcs = d.hcpcs
  WHERE d.adjudication_status = 'PAID'
    AND d.claim_type = 'DMEPOS'
    AND (d.modifier_1 = 'RR' OR d.modifier_2 = 'RR')
  GROUP BY d.beneficiary_id, ec.equipment_category, d.hcpcs, d.billing_npi,
           DATE_TRUNC('month', d.service_date)
),
sequenced AS (
  SELECT
      rm.*,
      ROW_NUMBER() OVER (PARTITION BY beneficiary_id, equipment_category
                         ORDER BY rental_month) AS rental_month_number,
      SUM(month_paid) OVER (PARTITION BY beneficiary_id, equipment_category
                            ORDER BY rental_month) AS cumulative_rental_paid
  FROM rental_months rm
)
SELECT
    s.beneficiary_id, s.equipment_category, s.hcpcs, s.supplier_npi,
    s.rental_month, s.rental_month_number,
    COALESCE(cap.cap_months, :default_rental_cap_months) AS cap_months,
    s.month_paid, s.cumulative_rental_paid,
    fs.purchase_price,
    ROUND(s.cumulative_rental_paid / NULLIF(fs.purchase_price, 0), 2) AS rental_to_purchase_ratio
FROM sequenced s
LEFT JOIN rental_cap_reference cap ON cap.equipment_category = s.equipment_category
LEFT JOIN fee_schedule fs          ON fs.hcpcs = s.hcpcs
WHERE s.rental_month BETWEEN :period_start AND :period_end
  AND s.rental_month_number > COALESCE(cap.cap_months, :default_rental_cap_months)
ORDER BY s.cumulative_rental_paid DESC;

-- rental_to_purchase_ratio > 1.0 means the payer has rented an item for more than it
-- would have cost to buy. That is a payment-policy conversation as much as a compliance
-- one, and it is a persuasive number to put in front of governance.
```

**4 · Threshold guidance.** :default_rental_cap_months = 13 reflects the common capped-rental period, but caps differ by
item category and by payer, so drive them from a reference table. The critical implementation
point is that the rental-month count must accumulate across the beneficiary's entire history
and across suppliers - a beneficiary who changes supplier mid-rental will otherwise restart the
count, which is exactly how caps are defeated in practice. Confirm your identification of
rental claims before trusting the sequence; the RR modifier is the usual marker but not a
universal one. Tune to your population; a starting point, not a standard.

**5 · Paired control + failure mode.** CB-08 (Quantity / frequency-limit edits - capped & time-based services). Secondary: CB-07 (MUE units-of-service edit enforcement), PA-07 (Continued-medical-necessity re-documentation for rentals/repairs).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No cumulative rental tracking. CB-08's frequency-limit exception report does not cover capped rentals. |
| **Designed-not-operating** | The edit exists but produced no exceptions in the period. |
| **Operated-but-bypassed** | Cap exceptions were overridden at adjudication without establishing the rental history. |
| **Operated-but-ineffective** | The count resets on a change of supplier or a change of HCPCS within the same equipment category, so the cap is trivially defeated. |
| **Evidence-missing** | No retained exception log for cap hits. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 11. Resupply before the item's reasonable useful lifetime

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
| **Secondary controls** | CB-08 - Quantity / frequency-limit edits (capped & time-based services); PA-07 - Continued-medical-necessity re-documentation for rentals/repairs … |

**1 · Data hypothesis.** Durable items carry a reasonable useful lifetime before replacement is payable, and consumable
supplies carry expected utilisation rates. Where the continued-need and recertification control
does not test replacement timing, items are replaced early and supplies are shipped faster than
they can be used - the auto-ship pattern, where resupply is driven by a shipping schedule
rather than by need.

**2 · What to check.** For durable items, flag replacements inside the reasonable useful lifetime with no documented
loss, theft or irreparable damage. For consumables, compute the shipment interval against
expected utilisation and flag suppliers whose resupply cadence systematically outruns it.

**3 · Reference SQL.**

```sql
-- TEST 11 - Early replacement and accelerated resupply
-- Params: :period_start, :period_end

-- 11a: durable item replaced inside its reasonable useful lifetime
SELECT
    'EARLY_REPLACEMENT' AS finding,
    new.claim_line_id, new.beneficiary_id, new.billing_npi AS supplier_npi,
    new.hcpcs, new.service_date, new.paid_amount,
    prior.service_date AS prior_supply_date,
    (new.service_date - prior.service_date) AS days_since_prior,
    rul.rul_months * 30 AS rul_days
FROM claim_lines new
JOIN equipment_category_map ec ON ec.hcpcs = new.hcpcs
JOIN reasonable_useful_life rul ON rul.equipment_category = ec.equipment_category
JOIN claim_lines prior
  ON  prior.beneficiary_id = new.beneficiary_id
  AND prior.hcpcs          = new.hcpcs
  AND prior.service_date   < new.service_date
  AND prior.adjudication_status = 'PAID'
WHERE new.service_date BETWEEN :period_start AND :period_end
  AND new.adjudication_status = 'PAID'
  AND (new.service_date - prior.service_date) < rul.rul_months * 30
  -- exclude documented loss / theft / irreparable damage replacements
  AND COALESCE(new.modifier_1,'') NOT IN ('RA') AND COALESCE(new.modifier_2,'') NOT IN ('RA')
  AND NOT EXISTS (SELECT 1 FROM replacement_justification j
                  WHERE j.claim_id = new.claim_id)

UNION ALL

-- 11b: consumable resupply faster than expected utilisation. The previous supply date is
-- looked up over a year before the period so the first in-period claim has a prior; only gaps
-- shorter than the expected supply interval are returned. Lines are first collapsed to one
-- row per beneficiary, code and service date: two lines for the same code on the same day
-- are one supply event, not a resupply after zero days.
SELECT
    'ACCELERATED_RESUPPLY',
    x.claim_line_id, x.beneficiary_id, x.billing_npi,
    x.hcpcs, x.service_date, x.paid_amount,
    x.prior_date,
    x.service_date - x.prior_date,
    x.expected_supply_days
FROM (
  SELECT d.*,
         LAG(d.service_date) OVER (PARTITION BY d.beneficiary_id, d.hcpcs
                                   ORDER BY d.service_date) AS prior_date
  FROM (
    SELECT cl.beneficiary_id, cl.hcpcs, cl.service_date,
           MIN(cl.claim_line_id)        AS claim_line_id,     -- first line of the day stands for the event
           MIN(cl.billing_npi)          AS billing_npi,
           SUM(cl.paid_amount)          AS paid_amount,       -- all lines of the day
           MIN(su.expected_supply_days) AS expected_supply_days
    FROM claim_lines cl
    JOIN supply_utilisation_reference su ON su.hcpcs = cl.hcpcs
    -- date minus integer days (PostgreSQL, DuckDB, Oracle); elsewhere use DATEADD(day, -365, start_date)
    WHERE cl.service_date BETWEEN :period_start - 365 AND :period_end
      AND cl.adjudication_status = 'PAID'
      AND cl.claim_type = 'DMEPOS'
    GROUP BY cl.beneficiary_id, cl.hcpcs, cl.service_date
  ) d
) x
WHERE x.service_date >= :period_start
  AND x.prior_date IS NOT NULL
  AND x.service_date - x.prior_date < x.expected_supply_days
ORDER BY finding, paid_amount DESC;

-- Companion roll-up the check asks for: by supplier, COUNT(*) and SUM(paid_amount) of the
-- ACCELERATED_RESUPPLY rows above, so a supplier-level pattern shows beside the single claims.
```

**4 · Threshold guidance.** Reasonable useful lifetime is set by policy and is not a tunable; expected utilisation for
consumables is, and it should come from the item's own labelled use rate rather than an
assumption. Two exclusions carry this test: documented loss, theft and irreparable damage
justify early replacement lawfully, and a change in the beneficiary's clinical condition
justifies a change of item. Build `replacement_justification` capture before running, or the
output will be dominated by legitimate replacements. For 11b, roll up to the supplier - a
single early shipment is a logistics artifact, a supplier whose whole book ships at 70% of the
expected interval has an auto-ship programme. Same-day lines for one code are treated as one
supply event and the expected interval is not scaled by the units supplied, so a double-
quantity shipment followed by a normal interval reads as on time. Tune to your population; a
starting point, not a standard.

**5 · Paired control + failure mode.** CD-10 (Continued-need / recertification documentation control for ongoing services, including DME rental). Secondary: CB-08 (Quantity / frequency-limit edits), PA-07 (Continued-medical-necessity re-documentation). See gap note G2 - no control names replacement timing or resupply cadence.

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No control tests replacement or resupply timing. CD-10's continued-need record does not address replacement intervals. |
| **Designed-not-operating** | CD-10 is per-interval; recertifications supporting replacement were not performed. |
| **Operated-but-bypassed** | Early replacements were approved on a supplier's assertion of damage with no supporting record. |
| **Operated-but-ineffective** | The control documents continued need for the item category but never tests whether a replacement was due, so early replacement passes a control designed for a different question. |
| **Evidence-missing** | No retained justification for replacements inside the useful lifetime. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 12. High-denial item categories with no documentation checklist in use

| | |
|---|---|
| **Paired control** | **CD-05** - Item/service-category documentation-eligibility checklist to prevent insufficient-documentation errors [open OIG rec] |
| **Control type** | Preventive |
| **Module** | Module 05 - Coverage & Documentation |
| **Owner** | Coding / PI |
| **Frequency** | Per claim |
| **System touchpoint** | Claim scrubber |
| **Evidence artifact** | Category eligibility checklist |
| **Control failure mode** | Category-specific insufficient-documentation overpayment (OIG catheters) |
| **Regulatory anchor** | CERT Program (Improper Payment Measurement) [case evidence]; OIG A-09-22-03019 (4 Feb 2025) - intermittent urinary catheters; rec 25-A-09-042.02 open, .01 and .03 closed [case evidence] |
| **Secondary controls** | PA-08 - Item-category documentation-eligibility checklists to prevent …; CD-15 - Targeted documentation education for high-risk item/service categories |

**1 · Data hypothesis.** Certain item categories generate insufficient-documentation errors repeatedly - the OIG
intermittent urinary catheter finding cited by both CD-05 and PA-08 is the current example.
Where the category documentation checklist is absent, the same documentation gap recurs
indefinitely. The signature is a category with a high and *stable* denial or error rate:
stability is what distinguishes a systemic documentation gap from a transient problem.

**2 · What to check.** Compute denial and post-payment error rates by item category over successive periods, identify
categories with persistently high rates, and reconcile against which categories actually have a
checklist deployed and education delivered.

**3 · Reference SQL.**

```sql
-- TEST 12 - Persistent category error rates versus checklist and education coverage
-- Params: :period_start, :period_end, :error_rate_cut (0.15), :min_claims (100)

WITH category_period AS (
  SELECT
      ec.item_category,
      DATE_TRUNC('quarter', cl.service_date) AS period,
      COUNT(*) AS claims,
      AVG(CASE WHEN cl.adjudication_status = 'DENIED'
                AND cl.denial_reason_group = 'INSUFFICIENT_DOCUMENTATION'
               THEN 1.0 ELSE 0 END) AS insufficient_doc_denial_rate,
      SUM(cl.paid_amount) AS paid_amount
  FROM claim_lines cl
  JOIN item_category_map ec ON ec.hcpcs = cl.hcpcs
  WHERE cl.service_date BETWEEN :period_start AND :period_end
  GROUP BY ec.item_category, DATE_TRUNC('quarter', cl.service_date)
  HAVING COUNT(*) >= :min_claims
),
persistent AS (
  SELECT item_category,
         COUNT(*) AS periods,
         AVG(insufficient_doc_denial_rate) AS mean_rate,
         MIN(insufficient_doc_denial_rate) AS min_rate,
         SUM(paid_amount) AS paid_amount
  FROM category_period
  GROUP BY item_category
  HAVING MIN(insufficient_doc_denial_rate) > :error_rate_cut     -- high in EVERY period
     AND COUNT(*) >= 3
)
SELECT
    p.item_category, p.periods,
    ROUND(p.mean_rate, 3) AS mean_insufficient_doc_rate,
    ROUND(p.min_rate, 3)  AS worst_case_floor,
    p.paid_amount,
    chk.checklist_id, chk.deployed_date,
    edu.last_education_date,
    CASE
      WHEN chk.checklist_id IS NULL AND edu.last_education_date IS NULL
        THEN 'NO_CHECKLIST_NO_EDUCATION'
      WHEN chk.checklist_id IS NULL THEN 'NO_CHECKLIST'
      WHEN edu.last_education_date IS NULL THEN 'NO_EDUCATION'
      ELSE 'BOTH_IN_PLACE_STILL_FAILING'
    END AS remediation_status
FROM persistent p
LEFT JOIN documentation_checklist chk ON chk.item_category = p.item_category
LEFT JOIN category_education edu      ON edu.item_category = p.item_category
ORDER BY p.paid_amount DESC;
```

**4 · Threshold guidance.** :error_rate_cut = 0.15 applied to the *minimum* across periods rather than the mean is the
design choice that matters here - requiring the rate to be high in every period filters out
categories having a bad quarter and isolates the ones with a structural documentation gap.
`remediation_status` is the finding: NO_CHECKLIST_NO_EDUCATION is a straightforward CD-05 and
CD-15 gap, while BOTH_IN_PLACE_STILL_FAILING is the more interesting result, because it means
the controls exist and do not work - an effectiveness question rather than a coverage one. Tune
to your population; a starting point, not a standard.

**5 · Paired control + failure mode.** CD-05 (Item/service-category documentation-eligibility checklist). Secondary: PA-08 (Item-category documentation-eligibility checklists for PA), CD-15 (Targeted documentation education for high-risk categories).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No category checklists exist. CD-05's category eligibility checklist does not exist. |
| **Designed-not-operating** | Checklists exist but are not applied at claim preparation for the affected categories. |
| **Operated-but-bypassed** | The checklist is completed as a formality without the underlying documentation being obtained. |
| **Operated-but-ineffective** | The checklist exists and the error rate has not moved - BOTH_IN_PLACE_STILL_FAILING - meaning the checklist does not reflect what reviewers actually require. |
| **Evidence-missing** | Checklists are used but not retained with the claim, so their use cannot be evidenced on review. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 13. Medical-necessity rationale absent at the coverage determination

| | |
|---|---|
| **Paired control** | **PA-06** - Reasonable-and-necessary documentation control per SSA 1862(a)(1)(A) at the medical-necessity determination |
| **Control type** | Preventive |
| **Module** | Module 04 - Prior Authorization |
| **Owner** | Reviewing clinician |
| **Frequency** | Per request |
| **System touchpoint** | Clinical review platform |
| **Evidence artifact** | Medical-necessity rationale record |
| **Control failure mode** | Payment for non-covered / unnecessary item |
| **Regulatory anchor** | SSA 1862(a)(1)(A) [binding]; Program Integrity Alliance - Value-for-Money in Anti-Fraud Efforts [case evidence] |
| **Secondary controls** | PA-03 - Complete PA package assembly & validation before submission (order + …; CD-03 - Reasonable-and-necessary documentation control: the medical record … |

**1 · Data hypothesis.** Where the reasonable-and-necessary documentation control at the determination point is absent,
prior authorisations are affirmed without a recorded clinical rationale. The consequence is
asymmetric and worth naming: it produces both improper payments (affirmations that should not
have been made) and improper denials (rejections with no articulated basis, which cannot be
meaningfully appealed).

**2 · What to check.** For each coverage determination, test for a recorded medical-necessity rationale and a named
reviewing clinician. Report by decision direction so affirmations and non-affirmations are
visible separately.

**3 · Reference SQL.**

```sql
-- TEST 13 - Determinations with no recorded medical-necessity rationale
-- SSA 1862(a)(1)(A). Params: :period_start, :period_end

SELECT
    r.pa_request_id, r.beneficiary_id, r.hcpcs, r.request_date,
    r.decision, r.decision_date,
    r.reviewing_clinician_npi,
    ra.rationale_id, ra.rationale_recorded_date,
    p.credential AS reviewer_credential,
    CASE
      WHEN ra.rationale_id IS NULL              THEN 'NO_RATIONALE_RECORDED'
      WHEN r.reviewing_clinician_npi IS NULL    THEN 'NO_NAMED_REVIEWER'
      WHEN p.credential IS NULL                 THEN 'REVIEWER_CREDENTIAL_UNKNOWN'
    END AS determination_defect,
    -- exposure differs by direction: affirmations carry payment risk,
    -- non-affirmations carry beneficiary-access risk
    CASE WHEN r.decision = 'AFFIRMED' THEN 'PAYMENT_RISK'
         ELSE 'ACCESS_RISK' END AS risk_direction,
    (SELECT COALESCE(SUM(cl.paid_amount), 0) FROM claim_lines cl
      WHERE cl.beneficiary_id = r.beneficiary_id
        AND cl.hcpcs = r.hcpcs
        AND cl.adjudication_status = 'PAID'
        AND cl.service_date BETWEEN r.decision_date AND r.decision_date + 365) AS paid_following
FROM pa_request r
LEFT JOIN pa_rationale ra ON ra.pa_request_id = r.pa_request_id
LEFT JOIN provider p      ON p.npi = r.reviewing_clinician_npi
WHERE r.decision_date BETWEEN :period_start AND :period_end
  AND (ra.rationale_id IS NULL
       OR r.reviewing_clinician_npi IS NULL
       OR p.credential IS NULL)
ORDER BY risk_direction, paid_following DESC;
```

**4 · Threshold guidance.** Categorical, and the `risk_direction` split is the reason to run it. An affirmation with no
rationale is a payment-integrity finding. A non-affirmation with no rationale is a beneficiary-
access finding, and it is the more serious of the two in its consequences for the person
involved - an unarticulated denial cannot be meaningfully appealed. Report both, and resist the
pull to work only the payment side because it is the one with a dollar figure attached. PA-19's
requirement that no denial rest solely on an algorithm is the adjacent control worth testing at
the same time. Tune to your population; a starting point, not a standard.

**5 · Paired control + failure mode.** PA-06 (Reasonable-and-necessary documentation control at the medical-necessity determination). Secondary: PA-03 (Complete PA package assembly & validation), CD-03 (Reasonable-and-necessary documentation).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No rationale is recorded at determination. PA-06's medical-necessity rationale record does not exist. |
| **Designed-not-operating** | PA-06 is per-request; a population of decisions carries no rationale. |
| **Operated-but-bypassed** | Rationale fields were completed with template text carrying no case-specific reasoning. |
| **Operated-but-ineffective** | A rationale is recorded but not by a clinician qualified to make the determination, which is the condition PA-19 addresses. |
| **Evidence-missing** | Determinations were reasoned but the reasoning is not retained in the case record. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 14. DMEPOS supplier standards - surety bond and accreditation currency

| | |
|---|---|
| **Paired control** | **PE-17** - DMEPOS surety bond & accreditation verification |
| **Control type** | Preventive |
| **Module** | Module 01 - Provider Enrollment |
| **Owner** | Enrollment specialist |
| **Frequency** | Per app/cycle |
| **System touchpoint** | NSC / accreditation data |
| **Evidence artifact** | Bond + accreditation on file |
| **Control failure mode** | Unaccountable DMEPOS suppliers |
| **Regulatory anchor** | 42 CFR 424.57(c),(d) - DMEPOS supplier standards, accreditation & $50,000 surety bond [binding]; SSA §1834(a)(16) [binding] |
| **Secondary controls** | PE-14 - Revalidation cycle enforcement + auto-deactivation; PE-02 - PECOS/enrollment-of-record data reconciliation cycle |

**1 · Data hypothesis.** DMEPOS suppliers must maintain accreditation and a surety bond as conditions of enrollment.
Where verification is performed once at enrollment and never revisited, suppliers bill while
unaccredited or unbonded - PE-17's stated failure mode is 'unaccountable DMEPOS suppliers', and
the bond exists precisely to make recovery possible when it goes wrong.

**2 · What to check.** For each supplier with paid DMEPOS claims, test accreditation status and surety bond currency
as at each service date, and quantify claims paid while either was lapsed.

**3 · Reference SQL.**

```sql
-- TEST 14 - Claims paid while accreditation or surety bond was lapsed
-- 42 CFR 424.57(c),(d); SSA §1834(a)(16). Params: :period_start, :period_end, :minimum_bond_amount
-- A supplier with no accreditation or bond record at all is labelled NO_..._RECORD. A supplier
-- that holds records, none of which covers the service date, is labelled
-- ACCREDITATION_NOT_IN_FORCE_ON_SERVICE_DATE or BOND_NOT_IN_FORCE_ON_SERVICE_DATE. Those two
-- labels say nothing about when cover stopped: it may have lapsed inside the period, years
-- earlier, or not yet have started. Read the earliest and latest claim dates beside them.

WITH acc_n AS (SELECT supplier_npi, COUNT(*) AS n FROM supplier_accreditation GROUP BY supplier_npi),
     bond_n AS (SELECT supplier_npi, COUNT(*) AS n FROM surety_bond GROUP BY supplier_npi)
SELECT
    s.supplier_npi, s.supplier_name_hash,
    acc.accreditation_org, acc.accreditation_start, acc.accreditation_end,
    bond.bond_amount, bond.bond_effective, bond.bond_expiry, bond.surety_name_hash,
    COUNT(cl.claim_line_id) AS claims_during_lapse,
    SUM(cl.paid_amount)     AS paid_during_lapse,
    MIN(cl.service_date)    AS earliest_lapsed_claim,
    MAX(cl.service_date)    AS latest_lapsed_claim,
    CASE
      WHEN acc.accreditation_org IS NULL AND COALESCE(an.n, 0) = 0 THEN 'NO_ACCREDITATION_RECORD'
      WHEN bond.bond_amount IS NULL AND COALESCE(bn.n, 0) = 0      THEN 'NO_BOND_RECORD'
      WHEN bond.bond_amount < :minimum_bond_amount                 THEN 'BOND_BELOW_MINIMUM'
      WHEN acc.accreditation_org IS NULL                           THEN 'ACCREDITATION_NOT_IN_FORCE_ON_SERVICE_DATE'
      ELSE 'BOND_NOT_IN_FORCE_ON_SERVICE_DATE'
    END AS supplier_standard_defect
FROM dmepos_supplier s
JOIN claim_lines cl
  ON  cl.billing_npi = s.supplier_npi
  AND cl.claim_type  = 'DMEPOS'
  AND cl.adjudication_status = 'PAID'
  AND cl.service_date BETWEEN :period_start AND :period_end
LEFT JOIN acc_n an ON an.supplier_npi = s.supplier_npi
LEFT JOIN bond_n bn ON bn.supplier_npi = s.supplier_npi
LEFT JOIN supplier_accreditation acc
       ON  acc.supplier_npi = s.supplier_npi
       AND cl.service_date BETWEEN acc.accreditation_start
                               AND COALESCE(acc.accreditation_end, DATE '9999-12-31')
LEFT JOIN surety_bond bond
       ON  bond.supplier_npi = s.supplier_npi
       AND cl.service_date BETWEEN bond.bond_effective
                               AND COALESCE(bond.bond_expiry, DATE '9999-12-31')
WHERE acc.accreditation_org IS NULL
   OR bond.bond_amount IS NULL
   OR bond.bond_amount < :minimum_bond_amount
GROUP BY s.supplier_npi, s.supplier_name_hash, acc.accreditation_org,
         acc.accreditation_start, acc.accreditation_end, bond.bond_amount,
         bond.bond_effective, bond.bond_expiry, bond.surety_name_hash, an.n, bn.n
ORDER BY paid_during_lapse DESC;

-- Companion worth running: suppliers sharing a surety_name_hash AND an address hash.
-- A single surety writing bonds for a cluster of co-located suppliers is a network
-- signal, and connects this test to family 5's ownership-graph work.
```

**4 · Threshold guidance.** :minimum_bond_amount = 50,000 per 42 CFR 424.57(d); confirm the current figure and any
category-specific variations. Certain supplier types are exempt from the bond requirement  - 
pharmacies and some professional practitioners among them - so build the exemption list before
running or those will dominate NO_BOND_RECORD. Join on the accreditation and bond records
effective for the *service date* rather than current status, or you will report every claim
from a supplier whose accreditation lapsed last month as retrospectively defective. Tune to
your population; a starting point, not a standard.

**5 · Paired control + failure mode.** PE-17 (DMEPOS surety bond & accreditation verification). Secondary: PE-14 (Revalidation cycle enforcement + auto-deactivation), PE-02 (PECOS/enrollment-of-record data reconciliation).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No bond or accreditation verification. PE-17's bond + accreditation on file artifact does not exist. |
| **Designed-not-operating** | PE-17 is per application/cycle; a population of active suppliers has no current verification. |
| **Operated-but-bypassed** | A lapse was identified and billing privileges continued pending renewal. |
| **Operated-but-ineffective** | Verification occurs at enrollment only and is never re-checked, so a lapse mid-cycle is invisible until revalidation - potentially three years. |
| **Evidence-missing** | Verification performed with no retained record of the bond or accreditation evidence reviewed. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 15. ABN not issued where non-affirmation or denial was likely

| | |
|---|---|
| **Paired control** | **PA-13** - ABN issuance when non-affirmation/denial is likely, before furnishing the item/service |
| **Control type** | Preventive |
| **Module** | Module 04 - Prior Authorization |
| **Owner** | Provider front-desk / supplier |
| **Frequency** | Per event |
| **System touchpoint** | ABN workflow |
| **Evidence artifact** | Signed ABN (CMS-R-131) |
| **Control failure mode** | Improper beneficiary liability / no advance notice |
| **Regulatory anchor** | CMS ABN (Form CMS-R-131) / SSA 1879 [binding] |
| **Secondary controls** | PA-06 - Reasonable-and-necessary documentation control per SSA 1862(a)(1)(A) …; PA-12 - Specific denial-reason communication + electronic PA (FHIR API) where … |

**1 · Data hypothesis.** This is the beneficiary-protection test in this family. An Advance Beneficiary Notice tells a
person, before an item is furnished, that it may not be covered and that they may become
liable. Where ABN issuance is absent, beneficiaries are billed after the fact for items they
had no notice were at risk. The predictable situations are visible in the data: item categories
and circumstances with high denial rates, furnished with no ABN.

**2 · What to check.** Identify furnished items in circumstances with a historically high denial rate - or where a
non-affirmation had already been issued - and test whether a signed ABN exists. Report the
beneficiary liability that resulted.

**3 · Reference SQL.**

```sql
-- TEST 15 - Items furnished without an ABN where denial was foreseeable
-- CMS Form CMS-R-131; SSA §1879. Params: :period_start, :period_end, :denial_rate_cut (0.30), :min_claims

WITH category_denial_rate AS (
  SELECT ic.item_category,
         AVG(CASE WHEN cl.adjudication_status = 'DENIED' THEN 1.0 ELSE 0 END) AS denial_rate,
         COUNT(*) AS claims
  FROM claim_lines cl
  JOIN item_category_map ic ON ic.hcpcs = cl.hcpcs
  WHERE cl.service_date BETWEEN :period_start - 365 AND :period_start - 1
  GROUP BY ic.item_category
  HAVING COUNT(*) >= :min_claims
),
foreseeable AS (
  SELECT
      cl.claim_line_id, cl.claim_id, cl.beneficiary_id, cl.billing_npi AS supplier_npi,
      cl.hcpcs, ic.item_category, cl.service_date,
      cl.adjudication_status, cl.beneficiary_liability_amount,
      cdr.denial_rate AS prior_year_category_denial_rate,
      CASE WHEN EXISTS (
        SELECT 1 FROM pa_request r
        WHERE r.beneficiary_id = cl.beneficiary_id
          AND r.hcpcs = cl.hcpcs
          AND r.decision = 'NON_AFFIRMED'
          AND r.decision_date < cl.service_date
      ) THEN 1 ELSE 0 END AS prior_non_affirmation
  FROM claim_lines cl
  JOIN item_category_map ic       ON ic.hcpcs = cl.hcpcs
  LEFT JOIN category_denial_rate cdr ON cdr.item_category = ic.item_category
  WHERE cl.service_date BETWEEN :period_start AND :period_end
    AND (COALESCE(cdr.denial_rate, 0) > :denial_rate_cut
         OR EXISTS (SELECT 1 FROM pa_request r2
                    WHERE r2.beneficiary_id = cl.beneficiary_id
                      AND r2.hcpcs = cl.hcpcs
                      AND r2.decision = 'NON_AFFIRMED'
                      AND r2.decision_date < cl.service_date))
)
SELECT
    f.*,
    abn.abn_id, abn.signed_date, abn.option_selected
FROM foreseeable f
LEFT JOIN abn_record abn ON abn.claim_id = f.claim_id
WHERE abn.abn_id IS NULL
ORDER BY f.prior_non_affirmation DESC, f.beneficiary_liability_amount DESC;
```

**4 · Threshold guidance.** :denial_rate_cut = 0.30 on the prior year's category denial rate is a starting definition of
'foreseeable', and it is a genuinely contestable one - a supplier is not obliged to anticipate
every denial, and issuing ABNs routinely as a blanket practice is itself improper. The stronger
and less arguable subset is `prior_non_affirmation = 1`: the item was already non-affirmed for
this beneficiary and furnished anyway with no notice. Work that population first. Note this
test's output is not an overpayment; it is beneficiary liability that may have been improperly
created, which is a different remedy and a different conversation. Tune to your population; a
starting point, not a standard.

**5 · Paired control + failure mode.** PA-13 (ABN issuance when non-affirmation/denial is likely, before furnishing the item/service). Secondary: PA-06 (R&N documentation at the medical-necessity determination), PA-12 (Specific denial-reason communication).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No ABN workflow. PA-13's signed CMS-R-131 does not exist. |
| **Designed-not-operating** | The workflow exists but no ABNs were issued for foreseeable-denial categories in the period. |
| **Operated-but-bypassed** | The item was furnished without notice and the beneficiary billed after denial. |
| **Operated-but-ineffective** | ABNs are issued as a blanket practice on every item, which is itself improper and provides no meaningful notice. |
| **Evidence-missing** | ABNs were issued but signed copies are not retained, so liability cannot be substantiated. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## 16. Items delivered during an inpatient or institutional stay

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
| **Secondary controls** | CD-09 - Proof-of-delivery / proof-of-service capture and retention with all …; PA-18 - Proof-of-delivery / service-vs-affirmation post-payment match |

**1 · Data hypothesis.** Equipment furnished to a beneficiary who is an inpatient is generally the facility's
responsibility and not separately payable, and equipment cannot be delivered to a home the
beneficiary is not living in. Where the setting coordination control is absent, DMEPOS claims
fall inside inpatient and skilled-nursing stays - sometimes a timing artifact, sometimes
billing for items that were never received.

**2 · What to check.** Interval-join paid DMEPOS claims against inpatient, SNF and hospice stays, excluding the
discharge day and the day before it, when delivery in anticipation of discharge is legitimate
and expected.

**3 · Reference SQL.**

```sql
-- TEST 16 - DMEPOS service dates inside an institutional stay
-- Params: :period_start, :period_end, :predischarge_grace_days (2)

SELECT
    d.claim_line_id, d.beneficiary_id, d.billing_npi AS supplier_npi,
    d.hcpcs, d.service_date, d.units, d.paid_amount,
    st.facility_type, st.facility_npi, st.admit_date, st.discharge_date,
    (d.service_date - st.admit_date) AS days_into_stay,
    (COALESCE(st.discharge_date, CURRENT_DATE) - d.service_date) AS days_before_discharge,
    ec.item_category
FROM claim_lines d
JOIN institutional_stay st
  ON  st.beneficiary_id = d.beneficiary_id
  AND d.service_date >  st.admit_date
  -- delivery in anticipation of discharge is legitimate: exclude the grace window
  AND d.service_date <  COALESCE(st.discharge_date, DATE '9999-12-31') - :predischarge_grace_days
LEFT JOIN item_category_map ec ON ec.hcpcs = d.hcpcs
WHERE d.service_date BETWEEN :period_start AND :period_end
  AND d.adjudication_status = 'PAID'
  AND d.claim_type = 'DMEPOS'
  AND st.facility_type IN ('INPATIENT_HOSPITAL','SNF','IRF','LTCH','HOSPICE')
  -- exclude item categories legitimately furnished during a stay
  AND NOT EXISTS (SELECT 1 FROM institutional_stay_carveout co
                  WHERE co.item_category = ec.item_category)
ORDER BY d.paid_amount DESC;
```

**4 · Threshold guidance.** :predischarge_grace_days = 2 is the parameter that matters most, and it should probably be
longer for items requiring fitting or home setup - a wheelchair or hospital bed delivered
several days before discharge is good discharge planning, not a finding. Set the grace window
by item category rather than globally. The carve-out list is equally load-bearing: prosthetics,
orthotics and certain supplies are legitimately furnished during a stay. `days_into_stay` is
the ranking column to use - an item dated in the middle of a long admission has fewer innocent
explanations than one dated near either boundary. Tune to your population; a starting point,
not a standard.

**5 · Paired control + failure mode.** CD-17 (Coordination-of-benefits / setting documentation control). Secondary: CD-09 (Proof-of-delivery capture and retention), PA-18 (POD / service-vs-affirmation post-payment match).

| If the hit is real, the control failed as… | …which means |
|---|---|
| **Absent** | No cross-setting check for DMEPOS. CD-17's COB documentation artifact does not cover equipment. |
| **Designed-not-operating** | CD-17 is per-claim; no setting check ran for DMEPOS claims in the period. |
| **Operated-but-bypassed** | Conflicts were identified at adjudication and paid on override. |
| **Operated-but-ineffective** | The check tests benefit-period overlap for services but excludes DMEPOS claim types from its scope entirely. |
| **Evidence-missing** | No retained record of setting conflicts and their disposition. |

**Result.** ☐ No hits  ☐ Hits - data-quality artifact  ☐ Hits - control finding  ☐ Hits - referred for investigation - _record the parameters used, the row count, and the tuned thresholds._

---

## Gap notes - controls this family needs that the library does not yet have

*Each note names a control objective these tests point to that the control library does not yet state. They are candidates for a future library release; the library itself is unchanged by this pack.*

### G1. Module 04 scope split between this family and family 8 - recorded so the boundary is deliberate.

Prior Authorization (Module 04) is covered by two families. This family uses the DMEPOS-
specific subset - PA-01, PA-02, PA-04, PA-05, PA-06, PA-07, PA-13 and PA-18 - because they are
inseparable from the DME order-to-encounter question. The general PA and claims-processing
controls (PA-09 through PA-12, PA-14 through PA-17, PA-19) are deliberately reserved for family
8, which covers prior-auth decision quality, timeliness, denial appropriateness and the PERM
data-processing component.

**Recommendation.** No action - this is a scoping note so a reader of both families understands the division and
does not read family 6 as complete coverage of Module 04. This family pairs 8 of the 19 Module
04 controls; family 8 covers the rest.

### G2. No control names replacement timing or resupply cadence.

Test 11 pairs to CD-10 (continued-need / recertification) because nothing closer exists, and
its 'Operated-but-ineffective' row records the mismatch: CD-10 documents continued need for an
item category but never asks whether a replacement was due. Reasonable useful lifetime and
expected supply utilisation are both bright-line, data-testable rules, and the auto-ship
resupply pattern - shipment driven by a schedule rather than by consumption - is a recurrent
and well-documented vulnerability with no owning control.

**Recommendation.** A replacement-and-resupply timing control for Module 05 or Module 04, stating both halves:
replacement inside the reasonable useful lifetime requires documented loss, theft or
irreparable damage; resupply requires evidence of consumption or continued need rather than an
elapsed shipping interval.

### G3. No control addresses beneficiary solicitation or telemarketing for DMEPOS.

Tests 1, 2 and 3 detect the *downstream* signature of order-generation schemes - orders with no
encounter, clinician fan-out, telehealth-originated orders. None of them touches the mechanism,
which is unsolicited contact with beneficiaries. PI-21 (beneficiary-side integrity analytics)
covers recruiter-clustered enrollment but not DME telemarketing, and the anti-solicitation
restrictions on DMEPOS suppliers are a distinct regulatory hook the library does not reference.

**Recommendation.** Candidate control for Module 01 or 02 covering supplier contact restrictions and lead-source
provenance. Note honestly that the analytics pairing would be weak: claims data cannot show
solicitation, and the only observable proxies are the downstream patterns already covered by
tests 1-3. A control here would be more valuable as an assessment question than as a query,
which is worth stating in its design rather than leaving a future analytics run to discover.

### G4. Test 13 surfaces the improper-denial mirror risk inside a payment-integrity family.

Test 13 splits determinations by `risk_direction`: an affirmation with no rationale is a
payment finding, a non-affirmation with no rationale is an access finding. This is the same
mirror risk family 4 raised as its G2, appearing here in a different module - and unlike
eligibility, Module 04 *does* have beneficiary-protection controls (PA-13 ABN issuance, PA-14
denial-appropriateness sampling, PA-15 external review of upheld denials, PA-19 no denial
solely by algorithm).

**Recommendation.** Worth noting as a positive precedent rather than a gap: Module 04 already models what family
4's G2 asks for on the eligibility side. When the beneficiary-protection control for
eligibility decisions is authored, PA-14 and PA-15 are the templates to follow, and this
family's test 13 shows the analytics pairing works - the same query serves both directions when
the output is split by decision direction.

---

*Reference patterns. Public-source, personal-capacity, advisory and organizational only. Queries must be adapted to your own schema and tested on your own warehouse before use. Outputs are review candidates for human adjudication, never payment or denial dispositions. Confirm every regulatory anchor against its primary source.*