[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.22003348.svg)](https://doi.org/10.5281/zenodo.22003348)

# HPI-CT — Healthcare Program Integrity Control Toolkit

**A free, open toolkit that helps healthcare payers, providers, and program-integrity agencies find and close the internal-control gaps that fraud, waste, and abuse exploit — *before* payment.** Built from the public enforcement record and aligned to the GAO Green Book (2025) and COSO.

*Created and maintained by **Nicholas Nyarko** — Chartered Accountant (CA), MBA, and healthcare financial analyst. Independent, personal-capacity, public-source work. Free to use. Attribution required.*

---

## Start here (no install)

Open `toolkit/HPI-CT_Toolkit.html` in any modern browser. It runs entirely offline — no server, no database, no account, nothing to install, and your answers never leave your browser (export them to a JSON file at any time; import brings them back or moves them to another machine).

Four jobs, from the home screen:

* **Assess** — set your Scope first (a short set of unscored questions that removes what was never yours to answer — nothing is removed until you answer), then work sub-area by sub-area. A **Core form (160 questions)** gives a screening read; the **Full form (246 questions)** is the instrument of record. Two scoring modes: self-reported, or evidence-validated, where each score is capped by how verifiable the evidence you name actually is.
* **Report** — a live module × component matrix with score, coverage and weight-at-risk lenses. Click any cell and it narrows to the exact questions behind that number, the controls a citation-backed link puts behind each gap, the requirement each control serves, and an owner-grouped evidence-request list you can copy and send. Set a baseline and a later report shows what moved.
* **Respond** — answer an audit or a finding: what they will ask for, the controls behind it, and a readiness line computed from your own answers.
* **Diagnose** — start from a suspicious signal in the data (58 signals, 29 flag families) and work back to the failed control and the questions that tell you if you are exposed.

Behind those sit the **controls library** (searchable, comparable, sortable by your own weakest answers), the **data-source registry**, the **case evidence**, a lighter **maturity model**, and the **knowledge graph**.

New to it? See `docs/HPI-CT_First_15_Controls_QuickStart.docx` for the fastest entry point, and `docs/HPI-CT_User_Guide.docx` for the full walkthrough.

## What's inside

* **172 controls** (78 preventive · 87 detective · 7 corrective; ~100 analytics-enabled) spanning the full operational cycle — governance, enrollment, eligibility, authorization, documentation, payment, post-payment, corrective action — across **10 modules** (9 control-bearing + governance).
* A **246-question, evidence-anchored self-assessment** mapped to the Green Book's five components and seventeen principles, with a 160-question Core screening form. Module 09 — **Financial Relationships & Referral Integrity** — covers the Anti-Kickback Statute, Stark, fair market value, MSO and marketing arrangements, EKRA, and beneficiary inducement.
* A **requirement registry of 84 cited objects** (statutes, regulations, subregulatory guidance and advisory findings, each classed by authority), a control-failure **diagnosis matrix**, and a **data-source registry** (18 sources with custodian, refresh cadence, match keys and caveats).
* **De-identified case exemplars** distilled from **1,586 coded U.S. DOJ healthcare-fraud enforcement actions** (2023–2026, tens of billions in alleged billings). The published tier in the toolkit carries **856 cases across 22 categories plus 9 state-derived patterns**, each mapped to the control that failed; the identified row-level base is held by the maintainer (see License).
* Machine-readable data in `/data` (JSON + CSV) — reuse it freely under CC BY 4.0.

## License — fully open

- **Code:** MIT ([`LICENSE`](LICENSE))
- **Content & data:** CC BY 4.0 ([`LICENSE-CONTENT`](LICENSE-CONTENT))

Use it, adapt it, deploy it, teach from it, build on it — commercially or not. The only obligation is **attribution**:

> HPI-CT — Healthcare Program Integrity Control Toolkit, created by Nicholas Nyarko, licensed under CC BY 4.0.

**Not included:** the row-level identified case dataset and raw intake are held by the maintainer; the exemplars here are de-identified. Researchers can request the identified source crosswalk under a short data-use understanding — nicholasnyarko@gmail.com.

## Cite this

> Nyarko, N. (2026). *HPI-CT: Healthcare Program Integrity Control Toolkit* (v1.0.0) [Framework and toolkit]. Zenodo. https://doi.org/10.5281/zenodo.22003348

## Contribute

Propose a control, fix a citation, suggest a case correction, or share how you used it — see **[`CONTRIBUTING.md`](CONTRIBUTING.md)**. Open an Issue or a Discussion.

## Using it? Tell us (optional)

Adoption helps this stay funded with attention and improve. If your organization uses HPI-CT, we'd value a one-line note or testimonial — open a Discussion or email nicholasnyarko@gmail.com. See **[`PRIVACY.md`](PRIVACY.md)** for exactly what the toolkit does and does not collect (short version: it runs offline and collects nothing unless you opt in).

## Scope & honesty

This is a **methodology and toolkit, advisory only** — it recommends controls; it never blocks or decides a payment, coverage, or eligibility action. Maturity scores are expert-judgment and self-reported. All case references are **allegations from public sources; defendants are presumed innocent**. See **[`SCOPE_AND_LIMITATIONS.md`](SCOPE_AND_LIMITATIONS.md)**. Confirm regulatory citations against primary `.gov` sources before relying on them.

*Not legal, compliance, or professional advice.*
