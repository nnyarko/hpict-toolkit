# Changelog

All notable changes to HPI-CT are recorded here.

This project publishes a [correction policy](https://github.com/nnyarko/hpict-toolkit) and holds
itself to it. Corrections appear here in plain language, including the ones that were our own
mistake — a release history that only records improvements is not one anyone should rely on.

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
  entirely. Anyone who took us up on "reuse it freely under CC BY 4.0" received a different
  library from the one the tool displays. `/data` is regenerated from the same source the toolkit
  is built from, so the two cannot diverge again.

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

### Note on how this happened

Module 09 was authored directly into the released toolkit file rather than into the source the
build pipeline reads. The tool was therefore correct and everything generated from the pipeline —
the published data, the counts in the README, the landing page — was a version behind. Module 09
has since been merged into the pipeline. The underlying cause was a build step bypassed under time
pressure, and it is recorded here because it is the kind of thing that recurs silently if it is
not named.

It did recur. Preparing this release we found that Module 09 was not the only thing authored
straight into the shipped HTML: a number of interface features were too, and the build pipeline
does not yet know about them. A rebuild therefore produces a toolkit **missing** features that
1.0.0 shipped. That is why 1.0.1 ships the 1.0.0 file unchanged rather than a rebuilt one — the
safe choice, and an honest admission that the pipeline is not yet the single source of truth this
project claims it should be.

The check that should have caught this could not have: it compared a fresh build against the
release folder, which the build had just overwritten. It could only ever pass. Parity is now
measured against the previously released artifact instead. Closing the gap between the pipeline
and the shipped file is tracked work and gates the next release, not this one.

**v1.0.0 remains available and is unchanged.** Zenodo versions rather than replaces, which is the
correct behaviour: the 2026-08-18 release is the record of what was published on that date. Cite
1.0.1 for the corrected data.

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
