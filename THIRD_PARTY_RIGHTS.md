# Third-party rights, sources, and what this project does and does not own

*Personal-capacity, public-source. **Not legal advice** — confirm with qualified counsel before relying on any of this.*

HPI-CT is built on other people's standards. Some of those are public domain and some are copyrighted, and the licence offered in `LICENSE-CONTENT` (CC BY 4.0) cannot grant rights in material this project does not own. This file states, source by source, what is being used and on what basis.

The short version: **the licence covers the original work — the control library, the assessment instrument, the coding rulebook, the case analysis, the toolkit and the documents. It does not, and cannot, cover the underlying standards.**

---

## 1. GAO Green Book — public domain, no restriction

**Standards for Internal Control in the Federal Government** (GAO-25-107721, 2025), including its five components and seventeen principles.

Works of the United States Government are not subject to copyright protection in the US (17 U.S.C. § 105). The Green Book may be quoted, reproduced, adapted, and built upon — including commercially — without permission.

**Consequence for this project:** the principle numbering and principle titles used throughout the control library and the assessment are sourced from the Green Book, not from COSO. This is deliberate, and it is the single most important choice in this document. The two frameworks share the same 5-component / 17-principle spine and the wording is close, but only one of them is free.

Attribution is given as a matter of scholarly practice. GAO asks that its work not be used in a way implying GAO endorsement; no endorsement is claimed or implied here.

## 2. COSO — copyrighted, referenced only

**Internal Control–Integrated Framework** and the **COSO/ACFE Fraud Risk Management Guide (2nd ed., 2023)** are copyrighted works of the Committee of Sponsoring Organizations of the Treadway Commission (and, for the FRMG, jointly with the ACFE). They are commercial publications.

**What this project does:** refers to COSO components and principles by name and number, and describes them in original wording, so that private-sector adopters — for whom COSO rather than the Green Book is the authoritative source — can locate the corresponding requirement. Facts, ideas, systems and methods of operation are not protected by copyright (17 U.S.C. § 102(b)), and citing a framework by its own reference numbers is ordinary professional practice across the audit literature.

**What this project does not do, and must not start doing:**

- reproduce COSO's framework text, principle statements verbatim at scale, or its "points of focus"
- reproduce the COSO cube or any COSO diagram
- redistribute any COSO or ACFE publication, in whole or in part
- describe itself as COSO-certified, COSO-approved, or COSO-compliant, or otherwise imply endorsement or affiliation

"COSO" and "ACFE" are the trademarks of their owners. They are used here descriptively, to identify the standards being mapped to — nominative use, not a claim of association.

**If in doubt, reference the Green Book instead.** Where a piece of wording must appear in the product, it should come from the public-domain source.

## 3. Federal regulatory and enforcement sources — public domain

42 CFR and the eCFR, CMS manuals (including the Program Integrity Manual, Pub. 100-08), HHS-OIG reports, compliance program guidance and the LEIE, GAO reports, and DOJ press releases and charging documents.

US Government works and edicts of government; not subject to copyright. Used freely, cited throughout.

## 4. The case base — original analysis of public records

Every coded case originates in a public government announcement or court record: DOJ press releases and takedown announcements, federal court opinions and dockets, HHS-OIG and GAO oversight findings, and state enforcement announcements.

The underlying facts are public and uncopyrightable. **What is original here — and what is licensed — is the coding**: the mapping of each matter to the control that failed, the lifecycle stage, the scheme category, the Green Book principle, and the selection and arrangement of the whole. That analytical layer is this project's work. Only the **de-identified** exemplars ship in this repository; the row-level identified dataset is held by the maintainer and available to researchers on request.

### On the Program Integrity Alliance

**No PIA-held data is in this project.** The internal case identifier prefix `PIA-NNN` is a naming convention only; it does not indicate a PIA source. The project's own run log records the PIA Connect lane as never connected across the entire harvest period, and every coded row carries a DOJ, court, OIG, GAO or state-enforcement source type.

PIA appears in two other ways, both ordinary: as an organisation this project has approached about collaboration, and as a cited author of a published policy paper (*Transparency by Design*), referenced in one control the way any source is referenced.

**If any PIA-held dataset is ever ingested, this changes.** A curated database can carry rights in its selection and arrangement even where the underlying facts are free, and in some jurisdictions a separate database right applies. Ingesting one requires written terms first, recorded here, before the data enters the case base — not after.

## 5. Professional bodies and taxonomies

NHCAA, AICPA, ACFE, HFPP and similar bodies are referenced as organisations, credential-granting authorities and landscape context. No proprietary taxonomy, curriculum or body of knowledge has been adopted or reproduced. The scheme categories used in the case base are this project's own descriptive categories, developed from the mechanics of the coded matters and documented in the Coding Rulebook.

## 6. Software dependencies

Chart.js (and any other vendored libraries) carry their own permissive open-source licences, which are preserved in the vendored files. They are not covered by this project's licences.

---

## What this means for licensing

| | |
|---|---|
| **Licensed by this project (CC BY 4.0 / MIT)** | the 160-control library and crosswalk, the assessment instrument and scoring model, the coding rulebook, the coded case analysis, the diagnosis matrix, the toolkit application, the white paper and all project documents |
| **Not licensable, because not owned** | the Green Book (public domain — nobody needs a licence), COSO and the COSO/ACFE FRMG (permission from COSO/ACFE if you need to reproduce them), federal regulations and enforcement records (public domain), vendored open-source libraries (their own licences) |

Anyone using HPI-CT under CC BY 4.0 receives rights in **this project's original work** (the only obligation is attribution). They do **not** receive, and this project cannot convey, any right to reproduce COSO's framework text or diagrams — that requires permission from COSO directly.

**The mapping is the product, not the standard.** What HPI-CT adds is the healthcare-specific operationalisation: which control, at which lifecycle stage, tested how, evidenced by what, against which failure observed in real enforcement. That is the part with a licence on it, and it is the part that took the work.

---

*Last reviewed 2026-08. Standards, terms of use, and the project's own sources change; this file should be revisited whenever a new source class is added.*
