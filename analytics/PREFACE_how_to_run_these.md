# Analytics Test Pack - how to run these on your warehouse, and the guardrails

*HPI-CT Healthcare FWA Prevention Framework · `analytics/` · reference patterns, public-source*

## What this pack is

The toolkit already has two analytics-adjacent layers. The **audit programs** tell you how
to test whether a control *operated*. The
**Diagnosis Matrix** tells you, when a flag fires, *which control broke*. Neither one ships
the query that produces the flag in the first place. This pack is that missing layer.

For each control in a family, it gives you five things:

1. **Data hypothesis** - what pattern the control's failure leaves in claims data.
2. **What to check** - the test in plain language.
3. **Reference SQL** - a dialect-neutral, parameterized pattern.
4. **Threshold guidance** - where to start, and why it is a start and not a standard.
5. **Paired control ID + failure mode** - the control to fix, mapped across the five
   failure modes: **Absent · Designed-not-operating · Operated-but-bypassed ·
   Operated-but-ineffective · Evidence-missing**.

Detection and diagnosis close a loop: the query surfaces a candidate, the failure-mode map
turns that candidate into a control finding, and the audit program tests the fix.

## The guardrails - read these before you run anything

**These queries produce review candidates. They never produce a payment or denial
decision.** Nothing in this pack should be wired to an automated denial, a payment
suspension, or a provider-network action. Every hit is a question for a human, and the
human's judgement is the control - not the query.

**The toolkit never receives your data.** These are patterns you run inside your own
environment against your own warehouse. No claim data, no PHI, and no identifiers leave your
systems at any point. That is a deliberate design property, not an oversight: it is what lets
this ship as a public, zero-integration artifact with no HIPAA surface of its own. Do not
send query *output* back to the project either - if you want to report on a pattern, report
counts and rates, never rows.

**Every query is a pattern, not a program.** They are written against generic claim-schema
field names. Yours will differ. Assume you will rewrite the FROM and JOIN clauses, remap
service categories to your own taxonomy, and check the unit-to-minute conversion against your
own fee schedule. A query that runs without error on your schema is not the same as a query
that is correct on your schema.

**Every threshold is a starting point.** Where a threshold is arithmetic (24 hours in a day,
a service date after a date of death) it is stated as such. Everywhere else the number is a
place to begin measuring your own distribution, not a standard to adopt. Tune to your
population. Publish the tuned value and the reason for it.

**Validate before you act.** Run each query first as a *count*, then look at twenty rows by
hand, then look at what the reciprocal or companion query says. Most first-run output in this
domain is a broken join key, not fraud - and the fastest way to lose credibility with a
provider network is to open a review on a data-quality artifact.

**Separate a control deficiency from an error.** A hit means a transaction looks wrong. The
failure-mode map tells you which control let it through. Those are two different findings with
two different remediations, and reporting them together is the most common way an analytics
program loses its argument.

**This is advisory and organizational.** Public-source, personal-capacity. It is not legal,
compliance, or actuarial advice, and it does not establish a standard of care. Confirm every
regulatory anchor against its primary source before operational use.

## Schema assumptions

The queries assume a conventional claims warehouse. Names below are the placeholders used
throughout; map them to yours once, at the top of each query.

| Placeholder | Meaning | Medicaid analogue (T-MSIS / TAF) |
|---|---|---|
| `claim_lines` | adjudicated claim-line fact table | TAF claim files (IP/LT/OT/RX), line level |
| `beneficiary_id` | enrollee identifier | `MSIS_ID` / TAF beneficiary link |
| `billing_npi` | entity that submitted the claim | `BLG_PRVDR_NPI` |
| `rendering_npi` | individual who performed the service | `SRVCNG_PRVDR_NPI` |
| `ordering_npi` | ordering / referring / prescribing provider | `RFRG_PRVDR_NPI`, ORP concepts |
| `hcpcs` | HCPCS / CPT procedure code | `PRCDR_CD` / line procedure code |
| `service_date` | date of service | `SRVC_BGN_DT` |
| `units` | billed units of service | `SRVC_QTY` |
| `paid_amount` | plan/state paid amount | `MDCD_PD_AMT` |
| `service_category` | your internal service grouping | derived from taxonomy + procedure code |

For Medicaid work, the T-MSIS/TAF data-quality picture matters as much as the schema - check
the DQ Atlas for the measures you are about to depend on before you trust a null rate:
<https://www.medicaid.gov/dq-atlas>. NCCI (including the Medicaid six-methodology
implementation referenced by control CB-06) is documented at
<https://www.cms.gov/medicare/coding-billing/ncci-medicaid>.

## Dialect notes

The SQL is written to be readable rather than portable, and leans on ANSI constructs
(CTEs, window functions, `LEAST`/`GREATEST`, `NULLIF`). Three things will need changing on
most warehouses:

- **Date and timestamp arithmetic.** `date_a - date_b` returns an integer in Postgres and
  Redshift, an interval for timestamps, a number of days in Oracle, and requires
  `DATEDIFF`/`DATE_DIFF`/`TIMESTAMPDIFF` in SQL Server, BigQuery and Snowflake.
- **`QUALIFY`.** Available in Snowflake, BigQuery and Teradata. Elsewhere, wrap the SELECT in
  a subquery and filter in the outer `WHERE`.
- **Named parameters.** Written as `:param`. Substitute your client's binding syntax
  (`?`, `@param`, `$1`). Do not string-concatenate values into these queries.

## How to work a family

1. Read the family's data hypotheses first, before any SQL. If a hypothesis does not describe
   a risk you actually carry, skip its test - running everything is not the goal.
2. Run the **reciprocal / companion** query noted under each test before the primary one.
   It tells you whether your join keys hold.
3. Run the primary query as `SELECT COUNT(*)` and check the order of magnitude against your
   own expectation. A test that returns 40% of your claim lines is measuring your schema, not
   your providers.
4. Sample twenty rows and read them.
5. Only then tune the threshold, and write down what you tuned it to and why.
6. Take the hits to the failure-mode map and turn them into control findings.

---

*The family documents are in `analytics/`; the same tests are in `data/analytics_tests.json` in
machine-readable form. All of them are reference patterns. Confirm every citation against its
primary source, and test every query on your own warehouse, before operational use.*
