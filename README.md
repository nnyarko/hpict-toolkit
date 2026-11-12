[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.22003347.svg)](https://doi.org/10.5281/zenodo.22003347)

# HPI-CT: Healthcare Program Integrity Control Toolkit

**A free, open toolkit that helps healthcare payers, providers, and program-integrity agencies find and close the internal-control gaps that fraud, waste, and abuse exploit, *before* payment.** Built from the public enforcement record, aligned to the GAO Green Book (2025) and cross-referenced to COSO by principle number.

*Created and maintained by **Nicholas Nyarko**: Chartered Accountant (CA), MBA, and healthcare financial analyst. Independent, personal-capacity, public-source work. Free to use. Attribution required.*

---

## Start here (no install)

Open **[`toolkit/HPI-CT_Toolkit.html`](toolkit/HPI-CT_Toolkit.html)** in any modern browser. It runs entirely offline: no server, no database, no account, nothing to install, and your answers never leave your browser (export them to a JSON file at any time; import brings them back or moves them to another machine).

Four jobs, from the home screen:

- **Assess**: set your Scope first (a short set of unscored questions that leaves out what was never yours to answer; a question that applies only if you answer Yes stays out until you do), then work sub-area by sub-area. A **Core form (160 questions)** gives a screening read; the **Full form (247 questions)** is the instrument of record. Two scoring modes: self-reported, or evidence-capped, where each score is capped by how verifiable the evidence you name actually is.
- **Report**: a live module × component matrix with score, coverage and weight-at-risk lenses. Click any cell and it narrows to the exact questions behind that number, the controls a citation-backed link puts behind each gap, the requirement each control serves, and an owner-grouped evidence-request list you can copy and send. Set a baseline and a later report shows what moved.
- **Respond**: answer an audit or a finding. See what they will ask for, the controls behind it, and a readiness line computed from your own answers.
- **Diagnose**: start from a suspicious signal in the data (58 signals, 29 flag families) and work back to the failed control and the questions that tell you if you are exposed.

Behind those sit the **controls library** (searchable, comparable, sortable by your own weakest answers), the **data-source registry**, the **case evidence**, a lighter **maturity model**, and the **knowledge graph**.

New to it? See **[`docs/HPI-CT_First_15_Controls_QuickStart.docx`](docs/HPI-CT_First_15_Controls_QuickStart.docx)** for the fastest entry point, and **[`docs/HPI-CT_User_Guide.docx`](docs/HPI-CT_User_Guide.docx)** for the full walkthrough. All four documents are listed in **[`docs/README.md`](docs/README.md)**.

## What's inside

- **173 controls** (79 preventive · 87 detective · 7 corrective; 101 analytics-enabled) spanning the full operational cycle (governance, enrollment, eligibility, authorization, documentation, payment, post-payment, corrective action) across **10 modules** (9 control-bearing + governance).
- A **247-question, evidence-anchored self-assessment** mapped to the Green Book's five components and seventeen principles, with a 160-question Core screening form. Module 09, **Financial Relationships & Referral Integrity**, covers the Anti-Kickback Statute, Stark, fair market value, MSO and marketing arrangements, EKRA, and beneficiary inducement.
- A **requirement registry of 88 cited objects** (statutes, regulations, subregulatory guidance and advisory findings, each classed by authority; control citations are labelled binding, guidance or case evidence), a control-failure **diagnosis matrix**, and a **data-source registry** (20 sources with custodian, refresh cadence, match keys and caveats).
- **De-identified case exemplars** distilled from **877 coded U.S. DOJ healthcare-fraud enforcement actions** (2023–2026; about $21.5 billion in alleged billings across the 555 that state a figure), each re-screened against a written inclusion rule (see `docs/How_the_case_base_is_scoped.md`). The published tier in the toolkit carries **871 cases across 21 categories plus 9 state-derived patterns**. Each of the 21 categories names the kind of control that failed in the case base's own shorthand (C-01 to C-10), which is not a library control ID; only the 9 state patterns link to controls in the library. The identified row-level base is held by the maintainer (see License).
- **128 reference detection tests** in 8 families, tied to library controls (`analytics/`, `data/analytics_tests.json`). They are patterns to adapt, tested only on invented data. No test covers the Module 09 (FR) controls.
- Machine-readable data in **[`/data`](data/)** (JSON + CSV), reuse it freely under CC BY 4.0 (the SQL inside `analytics_tests.json` is code, under MIT).

## License: fully open

- **Code:** MIT ([`LICENSE`](LICENSE)). SQL is code: the SQL in `data/analytics_tests.json` and `analytics/*.md` is under MIT.
- **Content & data:** CC BY 4.0 ([`LICENSE-CONTENT`](LICENSE-CONTENT))
- **Third-party libraries** embedded in the toolkit (Chart.js, @kurkle/color, Cytoscape.js) keep their own MIT licenses: [`THIRD_PARTY_LICENSES.md`](THIRD_PARTY_LICENSES.md).

Use it, adapt it, deploy it, teach from it, build on it, commercially or not. The only obligation is **attribution**:

> HPI-CT — Healthcare Program Integrity Control Toolkit, created by Nicholas Nyarko, licensed under CC BY 4.0.

**Not included:** the row-level identified case dataset and raw intake are held by the maintainer; the exemplars here are de-identified. Researchers can request the identified source crosswalk under a short data-use understanding, nicholasnyarko@gmail.com.

## Cite this

> Nyarko, N. (2026). *HPI-CT: Healthcare Program Integrity Control Toolkit* (v1.1.0) [Framework and toolkit]. Zenodo. https://doi.org/10.5281/zenodo.22003347

## Contribute

Propose a control, fix a citation, suggest a case correction, or share how you used it, see **[`CONTRIBUTING.md`](CONTRIBUTING.md)**. Open an Issue or a Discussion.

## Using it? Tell us (optional)

Adoption helps this stay funded with attention and improve. If your organization uses HPI-CT, we'd value a one-line note or testimonial, open a Discussion or email nicholasnyarko@gmail.com. See **[`PRIVACY.md`](PRIVACY.md)** for exactly what the toolkit does and does not collect (short version: it runs offline and collects nothing unless you opt in).

## Acknowledgements

Built from public records (DOJ, Free Law Project / CourtListener, HHS-OIG, GAO, CMS, and state enforcement sources) using public tools, including the Program Integrity Alliance's PIA Connect for discovery and harvest. Full credits in **[`ACKNOWLEDGEMENTS.md`](ACKNOWLEDGEMENTS.md)**. Acknowledgement is not endorsement.

## Scope & honesty

This is a **methodology and toolkit, advisory only**: it recommends controls; it never blocks or decides a payment, coverage, or eligibility action. Maturity scores are expert-judgment and self-reported. All case references are **allegations from public sources; defendants are presumed innocent**. See **[`SCOPE_AND_LIMITATIONS.md`](SCOPE_AND_LIMITATIONS.md)**. Confirm regulatory citations against primary `.gov` sources before relying on them. The case base was re-screened in October 2026 by AI agents working from official Justice Department text; this is internal checking, not independent review, and the counts are of detected and prosecuted matters with a claim on a payer.

*Not legal, compliance, or professional advice.*
