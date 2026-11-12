# Correction Policy

*HPI-CT: Healthcare Program Integrity Control Toolkit. Public-source, personal-capacity project maintained by Nicholas Nyarko. Effective 2026-07-29; last updated 2026-11-11.*

**Contact for all corrections: nicholasnyarko@gmail.com**: put "HPI-CT correction" in the subject line.

---

## 1. What this project studies

HPI-CT studies **control failures**, not people. The unit of analysis is the internal control that was missing or ineffective when a scheme succeeded, which lifecycle stage it belonged to, which COSO / GAO Green Book principle it maps to, and which control would close the gap.

Who was charged is not the subject and is not needed for the analysis. The published research therefore **does not identify defendants, respondents, or their businesses**.

## 2. What is published, and what is not

The project maintains two tiers, separated by a one-way de-identification step.

**Published tier: de-identified.** Scheme category, program, lifecycle stage, COSO/Green Book component and principle, module, source type, year, and the control-failure analysis. From every published row we remove:

- defendant names, case names, and party names, individual and corporate;
- citation URLs and docket numbers that resolve to a named party;
- exact dates (coarsened to year) and exact dollar figures (banded);
- the place value (district/state) wherever a place-plus-scheme combination covers fewer than three cases, since a rare combination can re-identify a case even with the name removed;
- any narrative note where an automated check cannot confirm that no name survives, such notes are withheld entirely rather than published on trust.

**Maintainer tier: identified, never published.** The full source records are retained privately so that duplicates can be detected, codings re-verified against primary sources, reliability studies run, and corrections acted on. This tier is excluded from the public repository and from version control. It is shared, if at all, only under a written data-use understanding for research verification.

Institutional sources are **not** party identifiers and are retained: GAO and OIG report numbers, CMS publications, and DOJ national-takedown or named-operation announcements. These are the government describing its own programs.

## 3. Presumption of innocence

Every enforcement matter in the underlying research is drawn from public announcements. An indictment, complaint, or charge is an **allegation**. Defendants are **presumed innocent unless and until proven guilty**. Aggregate dollar figures are *alleged* or *charged* amounts from charging documents, not adjudicated losses, and not amounts proven at trial.

## 4. What we correct

We will act on a report that any of the following is inaccurate:

- **Outcome changes**: an acquittal, dismissal, vacated conviction, overturned judgment, withdrawn charge, or a settlement that materially differs from what was announced.
- **A residual identification**: you believe a published row, note, or example still identifies you or your organization, whether directly or by combination of details.
- **A factual error** in a scheme classification, a control-failure characterization, a dollar figure, a date, or a citation.
- **A mis-citation**: a regulatory or case citation that does not support the statement attached to it.

The project also corrects itself. Release 1.1.0 records a correction the project found, not one reported to it: the case base counted matters that did not meet its own inclusion rule, and some control citations did not support what the control claimed. The CHANGELOG states what was wrong, the old and new figures side by side, and what did not change. Citations are now labelled binding, guidance or case evidence so a reader can see what kind of authority stands behind a control.

## 5. How to report, and what happens

Write to **nicholasnyarko@gmail.com**. Please include whatever you have: the sheet or artifact, any row identifier, and the primary source (court order, docket entry, official announcement) that establishes the correction. You do not need to be the affected party to report an error.

Our commitment:

| Step | Timeframe |
|---|---|
| Acknowledge receipt | **7 days** |
| Verify against primary sources and complete the correction | **30 days** of acknowledgement |
| Urgent removal where a report credibly alleges a live identification or reputational harm | **Immediate**, pending verification |

Where a report is credible but verification is still in progress, the disputed content is **withheld first and verified second**. We do not leave contested material published while we investigate.

When a correction is made we (a) correct or remove the underlying record in both tiers, (b) propagate the change through any derived statistic, learning, or published artifact, and (c) record the change (in `CHANGELOG.md`) describing what changed and when, without restating the identifying detail that prompted it.

If we conclude no correction is warranted, we will say so and explain why.

## 6. Limits

This is a personal-capacity project with no institutional backing, no legal department, and one maintainer. The commitments above are given in good faith and are what one person can honor; they are not a service-level agreement. This document is not legal advice, and the project is not affiliated with or endorsed by CMS, HHS-OIG, DOJ, GAO, COSO, or the ACFE.

Automated harvesting of new records is on hold as of release 1.1.0, and no records are being added. While it ran, records entered as candidates for review and the de-identification step ran before anything was published. Automation makes mistakes, which is exactly why this policy exists and why the contact address is monitored.

---

*See also: [`SCOPE_AND_LIMITATIONS.md`](SCOPE_AND_LIMITATIONS.md) for intended use and known limits, [`PRIVACY.md`](PRIVACY.md) for what the toolkit does with your answers, and [`CHANGELOG.md`](CHANGELOG.md) for the record of corrections made.*
