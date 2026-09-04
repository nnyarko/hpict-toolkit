# Changelog

All notable changes to HPI-CT are recorded here.

This project publishes a [correction policy](CORRECTION_POLICY.md) and holds itself to it.
Corrections appear here in plain language, including the ones that were the project's own mistake —
a release history that only records improvements is not one anyone should rely on.

---

## [1.0.2] — 2026-09-03

A transparency and documentation release. No change to the toolkit or the underlying data — the
controls, the questions, and the case coding are identical to 1.0.1. What changed is how the
project credits its sources and describes itself.

### Added

- **`ACKNOWLEDGEMENTS.md`.** A front-facing credit page for the public sources behind the case base
  and the controls — the U.S. Department of Justice, the Free Law Project / CourtListener, HHS-OIG,
  GAO, CMS, and state enforcement authorities — and for the Program Integrity Alliance's PIA Connect,
  used as a discovery and harvest layer to reach those public records. Standards and software are
  credited too. Acknowledgement is not endorsement.

### Fixed

- **`THIRD_PARTY_RIGHTS.md` corrected a factual error about PIA.** The prior text stated the PIA
  Connect lane was "never connected." That was wrong: PIA Connect was used to locate the public
  DOJ/OIG/GAO records that feed the case base. The section now describes that role accurately, while
  keeping the load-bearing point — the project codes the underlying primary public records, not any
  PIA-curated selection or arrangement.
- **External-facing rules reworded as commitments.** The COSO/ACFE section mixed a public compliance
  statement with a self-directed instruction ("must not start doing"). The public statement now
  reads as a commitment; the contributor-facing rule moved to `CONTRIBUTING.md`, where the people it
  addresses will see it.
- **Data files no longer labelled "DRAFT for review."** The export generator hard-stamped that
  status on every derived data file; released data now reads `"status": "released"`, and the
  generator is fixed so it stays that way.
- **`NOTICE` citation** updated from the v1.0.0 version DOI to the concept DOI, consistent with
  `CITATION.cff`.

Nothing here changes how the tool behaves or what the data contains.

---

## [1.0.1] — 2026-08-24

A correction release. No new content, and **the toolkit itself is unchanged** —
`toolkit/HPI-CT_Toolkit.html` is byte-identical to 1.0.0 (md5
`4a3dfea3a9244b081eca5fb79d223d50`). The tool was always right. What was wrong was everything
published *around* it: the machine-readable data, the README, the landing page and the document
set all described an earlier version.

Upgrading from 1.0.0 changes nothing about how the tool behaves.

### Fixed

- **The published data now matches the toolkit.** `/data/controls.json` shipped 160 controls and
  `/data/questionnaire.json` shipped 224 questions, while the toolkit itself contained 172 and 246.
  Module 09 — the newest and largest addition — was missing from the machine-readable data
  entirely. Anyone who reused that data under CC BY 4.0 received a smaller control library than
  the tool displays, and should re-download. `/data` is now generated from the same source the
  toolkit is built from, so the two cannot diverge again.

- **The README and the landing page described a user interface that did not exist.** Both promised
  "three modes — Assess, Look up, Why" and a "15-question triage". Those belong to a pre-release
  build. The shipped toolkit has four task cards — Assess, Report, Respond, Diagnose — a Scope
  step, sub-area working, and Core/Full forms. "Look up" was never in the released file. Anyone
  arriving at `index.html` was told to expect a search box and a triage, then handed something
  else. Both now describe the tool you actually get, with counts read from the generated
  statistics rather than typed in.

- **The landing page counts were a version behind the toolkit they linked to.** `index.html`
  advertised 160 controls, 224 questions, 9 modules and "1,550+" coded cases on the same page as a
  button opening a toolkit containing 172, 246, 10 and 1,586. Now generated from the same
  statistics file as everything else.

- **Duplicate and superseded documents removed.** The release contained two white papers, two user
  guides and two quick-starts — six files where three belong. One white paper was a July draft
  still marked "Draft v1.0 for review" and carrying a superseded contact address. A reader had no
  way to tell which was authoritative. Each document now appears once.

- **`CITATION.cff`** declared only the code licence; it now records the dual position (MIT for
  code, CC BY 4.0 for content and data) consistently with `LICENSE` and `LICENSE-CONTENT`. It also
  now cites the **concept DOI** rather than the v1.0.0 version DOI, so a citation resolves to the
  current release instead of pinning readers to a superseded one.

- **The white paper was fact-checked line by line and corrected.** Every statement of fact and
  every statistic was re-verified, against the project's own case base and exports or against a
  primary source. Substantive corrections:

  - The Operation Gold Rush passage reported that detection stopped all but ~$41M of the ~$4.45B
    scheduled for payment. That is accurate for Medicare, but the same charging documents put
    payments from Medicare **supplemental insurers** at nearly **$900 million**. The omission
    understated what the scheme actually collected by more than an order of magnitude.
  - The claim that the 2025 Green Book requires fraud-risk assessment "for the first time" was
    wrong — the 2014 edition already did, under Principle 8. Replaced with what the revision
    actually adds.
  - Counts that understated the project's own evidence base: oversight recommendations answered
    (23 → 35), oversight findings (110 → 175), court-record cases (72 → 156), requirement-registry
    objects (84 → 78), scheme categories (~25 → 89).
  - Medicaid's insufficient-documentation share is 77.2% and CHIP's is 56.1%; the paper had
    conflated the two. Shrank et al.'s fraud-and-abuse range is $58.5–83.9B, not a rounding.
  - Geographic concentration was restated from the data (Florida, Massachusetts, Texas, New Jersey,
    New York) with its coverage limit; the previous list was not supported.
  - Two claims were **removed** rather than softened, because neither could be evidenced: a
    market-sizing figure with no source, and an assertion that every control is "SMART" — no
    `achievable` or `time-bound` field exists on any control.
  - **No case in the base codes to Control Environment**, and the four populated components carry
    1,567 of 1,568 cases. Both are now stated as coding limitations rather than left for a reader
    to discover.
  - Table 1 is now prose. In v1.0.0 its cells were merged incorrectly, rendering "$26.97B" as
    "$26.97 | B 74%".
  - Every bracketed placeholder is gone, including one that named a manuscript under review at a
    named venue — which had no business in a permanently citable document.

### Cause

Module 09, and a number of interface features, were authored directly into the released toolkit
file rather than into the source the build pipeline reads. The tool was therefore correct while
everything generated from the pipeline — the published data, the counts in the README and on the
landing page — described an earlier version.

Because the pipeline does not yet know about those interface features, a rebuild produces a
toolkit *missing* functionality that v1.0.0 shipped. This release therefore publishes the v1.0.0
file unchanged rather than a regenerated one. Reconciling the pipeline with the shipped file is
the priority for the next release.

The parity check that should have caught the divergence could not have: it compared a fresh build
against the release folder that the build had just overwritten, so it could only ever pass. Parity
is now measured against the previously released artifact.

### Known limitation carried by v1.0.0

The `/data` files published with v1.0.0 contain 160 controls and 224 questions against the
toolkit's 172 and 246. That version remains available and unchanged — Zenodo versions rather than
replaces, and the 2026-08-18 record is the correct record of what was published on that date — but
anyone who reused its data should upgrade. Cite v1.0.1 for the corrected data, or the concept DOI
`10.5281/zenodo.22003347` to always resolve to the current release.

---

## [1.0.0] — 2026-08-18 (first open release)

- **Fully open licensing:** MIT (code) + CC BY 4.0 (content & data). Free to use, adapt, and
  deploy with attribution.
- **Task-first toolkit.** The home screen asks what you need to do and offers four jobs — *Assess*,
  *Report*, *Respond*, *Diagnose* — rather than a row of tabs. Behind them sit the controls
  library, data-source registry, case evidence, maturity model and knowledge graph.
- **Scope before assessment.** A short set of unscored questions removes what was never yours to
  answer. Nothing is descoped until you answer it, and every screen that shows a reduced total
  prints the arithmetic behind it.
- **Sub-area working.** Each module breaks into sub-areas of a few questions, independently scored,
  so the instrument can be worked in short sessions and split across people.
- **Evidence-validated scoring.** Name what an answer rests on — oral, documentary, or
  system-generated — and the score is capped by how verifiable that is. The cap is shown before you
  answer and named in the result, so the ladder teaches rather than hides.
- **Core and Full forms.** 160 questions for a screening read; 246 for the instrument of record.
  The report names which form produced it.
- **172 controls** (78 preventive · 87 detective · 7 corrective) across 10 modules (9
  control-bearing + governance), including **Module 09 — Financial Relationships & Referral
  Integrity** covering the Anti-Kickback Statute, Stark, fair market value, MSO and marketing
  arrangements, EKRA and beneficiary inducement.
- **246-question** evidence-anchored assessment; requirement registry of 84 cited objects classed
  by authority; control-failure diagnosis matrix; data-source registry of 18 sources.
- **De-identified case exemplars** distilled from 1,586 coded DOJ enforcement actions; 856 in the
  published tier across 22 categories plus 9 state-derived patterns. The identified row-level
  dataset is held by the maintainer and available to researchers on request.
- **Runs offline.** One self-contained HTML file, no server, no account, no network call after
  load. Answers stay in your browser; export and import move them between machines.
- Machine-readable exports (JSON + CSV) for controls, rules, data sources, diagnosis matrix,
  questionnaire and case exemplars.
- Citable release with a DOI, version stamped in the header, the report footer and every export.

_The row-level identified case dataset and raw intake are not part of this release; they remain
with the maintainer and are available to researchers on request._

---

## Known limitations carried into 1.0.1

Stated here rather than discovered later:

- **Module 09 is hand-authored and not yet subject-matter reviewed.** Its regulatory anchors and
  the $15/$75 nominal-value thresholds are flagged for verification against primary sources, and
  the option keying on its 22 questions has not had an independent read. The flag travels with the
  data: every Module 09 record in `/data` carries `provenance` and `verification` fields.
- **The assessment instrument has not been validated.** There is no inter-rater reliability study;
  two people scoring the same organisation may differ, and by how much is unmeasured.
- **Control-to-case mappings are design hypotheses, not proven interceptions.** That a control
  addresses a failure mode observed in a case is not a claim that it would have prevented that
  case. See `SCOPE_AND_LIMITATIONS.md`.
