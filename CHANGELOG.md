# Changelog

All notable changes to HPI-CT are recorded here.

This project publishes a [correction policy](CORRECTION_POLICY.md) and holds itself to it.
Corrections appear here in plain language, including the ones that were the project's own mistake, 
a release history that only records improvements is not one anyone should rely on.

---

## [1.1.0] - 2026-11-11

A correction and extension release. The case base was re-screened against an inclusion rule that earlier versions never stated, and the
number of cases counted fell from 1,586 to 877. (1,586 was the count published with versions 1.0.0 to 1.0.2. The harvest went on adding
rows after that count was taken, so the base that was re-screened held 1,605 rows, and `docs/How_the_case_base_is_scoped.md` accounts for
all 1,605.) This was the project's own mistake: the case base was built by keyword harvest and its rows
were never checked against the rule that a case must concern health care and a claim on a payer. This release says so and shows the working.

### Changed

- **Case base re-screened.** All 1,605 rows were put through re-screening against the official Justice Department text under four tests (health care; a claim on a
  payer; one matter in its own source; Justice Department source, 2023 or later). 877 rows are counted. 239 rows are the same matter as another
  row (a plea and a later sentencing, co-defendants of one scheme, the same release on two pages) and were merged. 151 were
  controlled-substance prescribing or diversion with no claim on a payer; 97 were pandemic relief fraud; 78 were announcements or round-ups;
  63 were not health care; others are listed in `docs/How_the_case_base_is_scoped.md`. 39 unresolved rows are kept on an uncounted watch list.
  6 rows could not be matched to an official page or could not be read; they are marked not read yet and are not counted, so 1,599 of
  the 1,605 rows were read. No row is deleted.
- **All case figures recomputed** from the counted rows. Counts fell in every scheme category, most in COVID-19 testing (117 to 21) and opioid
  and controlled-substance matters (253 to 66). The per-category dollar totals now follow a written method (counted rows only, rows with a
  stated figure only, largest row shown, rows of $1B or more listed).
- **Coding rulebook v1.2.** Every rule carries its reason; a rule for treating a plea and a sentencing as one matter; definitions of the
  categories that were being misused; a recorded screening step and a quarterly re-read of 50 rows.
- **Citations corrected and labelled.** The October 2026 citation check rated 11 items wrong: 10 control citations and one registry row.
  All 11 were corrected against the primary source, and so were 7 other control citations rated partly supported, so 17 controls and one
  registry row changed (for example an OIG report number
  that did not exist, a regulation cited for something it does not say, and a GAO report cited for a control it does not concern). A further pass on 11 November 2026 corrected the partly supported citations whose correction was clear from the regulation's current text (among them CD-09, PE-03, FR-02, FR-11, CD-08 and REG-001); 152 partly supported items were left as they are, each with a recorded reason, so a citation may still be broader or narrower than the control states. Control
  citations carry a label: binding (statute or regulation), guidance, or case evidence. A few notes and cross-references between controls are
  not labelled.
- **Green Book principles shown with their names** everywhere a principle appears, instead of a bare number. The 2025 revision of the Green
  Book was checked: it keeps the same 17 principles in the same order; principle 8 now also covers improper payment and information security.
- **Graph screen rewritten in plain language,** with every layer rebuilt for the current library (several had been built for 160 controls).
- **Diagnose screen.** 14 signal families named controls by an old numbering that is not in the library; they now point to library controls,
  or say that no control in the library does that job.
- **Graph screen links re-derived.** The links between questions and controls on the Graph screen had been derived by a step that ignored
  the wording of each question and compared only module, principle number and sub-area heading. The defect was fixed and the links were
  derived again. Before the fix, 33 controls showed no linked question and 87 showed strong links; after it, 157 show strong links, 9 moderate and 7 none (PE-17, PI-21, PI-22, PI-23, MD-22, MD-24 and MD-26). A manual read found those 7 to be real gaps in the questionnaire, as are three of the 9 moderate ones (PI-20, MC-22, MD-23). The links remain machine-derived and have not been reviewed by a person.
- **Published exemplars.** The published tier in the toolkit went from 856 cases in 22 categories to 871 cases in 21 categories, plus the
  same 9 state-derived patterns. The category removed is ACA enrollment fraud, which has no counted row left. The other 6 of the 877
  counted rows sit in two small categories that have no entry in the published tier (mental health / psychotherapy, 5; autism / ABA
  therapy, 1).
- **Source type on kept rows.** Where several rows were merged into one matter, the row kept now carries the latest criminal stage the
  matter reached (for example a plea that was later sentenced is shown as a sentencing). This changed the source type on 72 kept rows. The
  count of 877 is unchanged.
- **Redaction defect fixed.** The step that removes party names had treated some ordinary words as names, so the de-identified case
  workbook showed "[party removed]" inside source-type and component labels. The defect predates this release. The labels are now on a
  keep list and show correctly.
- **Questionnaire text edits.** The source reference shown under two questions (Module 05 A2 and Module 07 H1) was corrected to follow
  the citation corrections, and a working-status marker was removed from the questionnaire file's note. Apart from the new MC-29 question,
  no question wording, answer option or score changed.
- **Open-recommendation flags re-checked.** The flags that mark a control as resting on an open oversight recommendation were re-checked
  against the status the issuing body publishes. Flags cleared because the recommendation was found closed: nine (PI-04, PA-09, PA-10, CD-15, CD-16, CD-18, MD-09, MD-17 and MC-27). Two controls gained the flag (CB-09 and CB-13), and 25 controls now carry it. Status was read at the GAO or OIG report page on 11 November 2026, except MC-27, which rests on the October check.
- **Build pipeline reconciled with the shipped toolkit.** Release 1.0.1 said the build pipeline did not know about Module 09 and several
  interface features, and named reconciling the two as the priority for the next release. That is now done: the toolkit in this release is
  generated by the pipeline.
- **Credits and licenses.** Third-party libraries are now credited by name and version, with their copyright lines and the MIT text in
  `THIRD_PARTY_LICENSES.md`. The README and `data/README.md` now say that SQL is code and is under MIT. `LICENSE` now opens with the MIT
  text. A notice on the CPT trademark, credits for further public standards, and a disclosure of AI assistance were added to
  `ACKNOWLEDGEMENTS.md` and the citation metadata.

- **Scoring mode renamed.** The mode formerly labelled "evidence-validated" is now "evidence-capped", because what it does is cap a score by how verifiable the named evidence is. The scoring itself is unchanged.
- **Wording and sources.** Rule notes no longer carry a "Draft" marker; the detection-test files and the Graph screen no longer describe themselves as drafts; the improper-payment weighting file now states what was checked (the program rates and amounts, against the CMS FY2025 fact sheet on 2026-11-05) and what is judgment (the cause weights); the Scope document no longer says regulation citations were verified, and now gives the result of the October 2026 citation check (132 of 313 items fully supported on the first pass).
- **Documents.** The white paper, quick start and user guide were updated to the new counts, and the white paper was corrected where it was wrong or overstated: the 2025 Green Book's improper-payment and information-security requirement sits in principle 8, not principle 7; the requirement registry has 88 objects, not 78; the Gold Rush section is now called a walkthrough, not a validation; claims about oversight recommendations, pilots and ongoing monitoring were reduced to what can be supported. The README, quick start and user guide now cite the concept DOI.
- **Scope wording corrected.** The Scope screen, README, user guide and white paper said nothing is removed until you answer. In fact a question that applies only if you answer Yes (for example Medicare Advantage, managed care, DMEPOS or the AI flags) stays out until you do, so before you answer, the assessment lists about 217 of the 247 questions (152 of the 160 on the Core form). The wording now says so. Scoring is unchanged.
- **Scope screen behaviour.** No answer is selected in advance: each Scope question shows as unanswered until you answer it. Clicking a chosen answer again clears it. A Next unanswered button moves to the next open question. The sidebar counts follow the form you chose (the Core count leaves out Deep questions) and are labelled a starting count until the Scope questions are answered. On the Assess screen too, clicking a chosen answer again clears it, so any question can go back to unanswered.

### Added

- **New control MC-29, Preclusion List screening before payment** (Medicare Advantage and Part D), with its rule row (42 CFR 422.222), its data
  source link (the CMS Preclusion List) and a new assessment question. The gap was found by a completeness check: the source was in the data
  source registry with no control behind it. The library is now 173 controls and the assessment 247 questions.
- **Registry additions.** Two data sources (the National Practitioner Data Bank; state Medicaid exclusion and termination lists), linked to the
  state-and-federal list screening control PE-09. Three rule rows: the CMS Medicaid and CHIP Program Integrity Pillars (guidance), and the two
  Medicaid death-file provisions of Public Law 119-21 (sections 71104 and 71105). The registries now hold 20 data sources and 88 rules.
- **Detection tests.** 128 reference detection patterns in 8 families, each tied to library controls, shown on each control's screen with the
  SQL, a threshold note, and the failure mode a hit points to. They are reference patterns to adapt, never an automated decision. They were run
  on invented data only, and a first run on 6 October 2026 found 40 of the 128 with a known defect (39 failed and one did not run as written). Those defects were fixed and the pack was run again on 10 November 2026. A further review by AI readers (internal checking, not independent review) then found more logic errors in situations the invented data does not exercise, and those tests were fixed again. On the final run on 11 November 2026, 127 of the 128 flagged every planted case and none of the named look-alikes on the invented dataset, and one cannot be tested on claims data alone. Known open points are listed in each test's notes. Each result is shown on screen beside its test. No test covers the Module 09 (FR) controls. `data/analytics_tests.json`; `analytics/`.
- `docs/How_the_case_base_is_scoped.md`, and `docs/README.md` listing the four documents.
- `THIRD_PARTY_LICENSES.md`.

### Notes

The scoring and parity with the production toolkit are unchanged for the questions both versions carry (checked). The screening was done by AI
agents on one model and checked internally; it is not independent review. A first re-read of 50 counted rows found no inclusion failure but
about one category label in five clearly wrong (9 of 50); the categories, stages and components have not been re-coded, so per-category
figures are approximate. No dollar figure or count in the earlier releases should be quoted next to the new ones.

`SCOPE_AND_LIMITATIONS.md` now discloses, for the first time: an internal hold-out test whose result fell below the line set beforehand
(33 of 62 cases, 53 percent, against a 60 percent line); four control gaps the tests found; that no detection test covers the Module 09
controls; and that the best-scoring answer is listed first in 165 of the 247 questions.

---

## [1.0.2] - 2026-09-03

A transparency and documentation release. No change to the toolkit or the underlying data, the
controls, the questions, and the case coding are identical to 1.0.1. What changed is how the
project credits its sources and describes itself.

### Added

- **`ACKNOWLEDGEMENTS.md`.** A front-facing credit page for the public sources behind the case base
  and the controls: the U.S. Department of Justice, the Free Law Project / CourtListener, HHS-OIG,
  GAO, CMS, and state enforcement authorities, and for the Program Integrity Alliance's PIA Connect,
  used as a discovery and harvest layer to reach those public records. Standards and software are
  credited too. Acknowledgement is not endorsement.

### Fixed

- **`THIRD_PARTY_RIGHTS.md` corrected a factual error about PIA.** The prior text stated the PIA
  Connect lane was "never connected." That was wrong: PIA Connect was used to locate the public
  DOJ/OIG/GAO records that feed the case base. The section now describes that role accurately, while
  keeping the load-bearing point: the project codes the underlying primary public records, not any
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

## [1.0.1] - 2026-08-24

A correction release. No new content, and **the toolkit itself is unchanged**, 
`toolkit/HPI-CT_Toolkit.html` is byte-identical to 1.0.0 (md5
`4a3dfea3a9244b081eca5fb79d223d50`). The tool was always right. What was wrong was everything
published *around* it: the machine-readable data, the README, the landing page and the document
set all described an earlier version.

Upgrading from 1.0.0 changes nothing about how the tool behaves.

### Fixed

- **The published data now matches the toolkit.** `/data/controls.json` shipped 160 controls and
  `/data/questionnaire.json` shipped 224 questions, while the toolkit itself contained 172 and 246.
  Module 09 (the newest and largest addition) was missing from the machine-readable data
  entirely. Anyone who reused that data under CC BY 4.0 received a smaller control library than
  the tool displays, and should re-download. `/data` is now generated from the same source the
  toolkit is built from, so the two cannot diverge again.

- **The README and the landing page described a user interface that did not exist.** Both promised
  "three modes: Assess, Look up, Why" and a "15-question triage". Those belong to a pre-release
  build. The shipped toolkit has four task cards (Assess, Report, Respond, Diagnose) a Scope
  step, sub-area working, and Core/Full forms. "Look up" was never in the released file. Anyone
  arriving at `index.html` was told to expect a search box and a triage, then handed something
  else. Both now describe the tool you actually get, with counts read from the generated
  statistics rather than typed in.

- **The landing page counts were a version behind the toolkit they linked to.** `index.html`
  advertised 160 controls, 224 questions, 9 modules and "1,550+" coded cases on the same page as a
  button opening a toolkit containing 172, 246, 10 and 1,586. Now generated from the same
  statistics file as everything else.

- **Duplicate and superseded documents removed.** The release contained two white papers, two user
  guides and two quick-starts: six files where three belong. One white paper was a July draft
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
    wrong: the 2014 edition already did, under Principle 8. Replaced with what the revision
    actually adds.
  - Counts that understated the project's own evidence base: oversight recommendations answered
    (23 → 35), oversight findings (110 → 175), court-record cases (72 → 156), requirement-registry
    objects (84 → 78), scheme categories (~25 → 89).
  - Medicaid's insufficient-documentation share is 77.2% and CHIP's is 56.1%; the paper had
    conflated the two. Shrank et al.'s fraud-and-abuse range is $58.5–83.9B, not a rounding.
  - Geographic concentration was restated from the data (Florida, Massachusetts, Texas, New Jersey,
    New York) with its coverage limit; the previous list was not supported.
  - Two claims were **removed** rather than softened, because neither could be evidenced: a
    market-sizing figure with no source, and an assertion that every control is "SMART", no
    `achievable` or `time-bound` field exists on any control.
  - **No case in the base codes to Control Environment**, and the four populated components carry
    1,567 of 1,568 cases. Both are now stated as coding limitations rather than left for a reader
    to discover.
  - Table 1 is now prose. In v1.0.0 its cells were merged incorrectly, rendering "$26.97B" as
    "$26.97 | B 74%".
  - Every bracketed placeholder is gone, including one that named a manuscript under review at a
    named venue, which had no business in a permanently citable document.

### Cause

Module 09, and a number of interface features, were authored directly into the released toolkit
file rather than into the source the build pipeline reads. The tool was therefore correct while
everything generated from the pipeline: the published data, the counts in the README and on the
landing page: described an earlier version.

Because the pipeline does not yet know about those interface features, a rebuild produces a
toolkit *missing* functionality that v1.0.0 shipped. This release therefore publishes the v1.0.0
file unchanged rather than a regenerated one. Reconciling the pipeline with the shipped file is
the priority for the next release.

The parity check that should have caught the divergence could not have: it compared a fresh build
against the release folder that the build had just overwritten, so it could only ever pass. Parity
is now measured against the previously released artifact.

### Known limitation carried by v1.0.0

The `/data` files published with v1.0.0 contain 160 controls and 224 questions against the
toolkit's 172 and 246. That version remains available and unchanged: Zenodo versions rather than
replaces, and the 2026-08-18 record is the correct record of what was published on that date, but
anyone who reused its data should upgrade. Cite v1.0.1 for the corrected data, or the concept DOI
`10.5281/zenodo.22003347` to always resolve to the current release.

---

## [1.0.0] - 2026-08-18 (first open release)

- **Fully open licensing:** MIT (code) + CC BY 4.0 (content & data). Free to use, adapt, and
  deploy with attribution.
- **Task-first toolkit.** The home screen asks what you need to do and offers four jobs, *Assess*,
  *Report*, *Respond*, *Diagnose*: rather than a row of tabs. Behind them sit the controls
  library, data-source registry, case evidence, maturity model and knowledge graph.
- **Scope before assessment.** A short set of unscored questions removes what was never yours to
  answer. Nothing is descoped until you answer it, and every screen that shows a reduced total
  prints the arithmetic behind it.
- **Sub-area working.** Each module breaks into sub-areas of a few questions, independently scored,
  so the instrument can be worked in short sessions and split across people.
- **Evidence-validated scoring.** Name what an answer rests on, oral, documentary, or
  system-generated, and the score is capped by how verifiable that is. The cap is shown before you
  answer and named in the result, so the ladder teaches rather than hides.
- **Core and Full forms.** 160 questions for a screening read; 246 for the instrument of record.
  The report names which form produced it.
- **172 controls** (78 preventive · 87 detective · 7 corrective) across 10 modules (9
  control-bearing + governance), including **Module 09: Financial Relationships & Referral
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

## Known limitations carried into 1.1.0

First stated with 1.0.1 and still true. The fuller list is in `SCOPE_AND_LIMITATIONS.md`.

- **Module 09 is hand-authored and not yet subject-matter reviewed.** Its regulatory anchors and
  the $15/$75 nominal-value thresholds are flagged for verification against primary sources, and
  the option keying on its 22 questions has not had an independent read. The flag travels with the
  data: every Module 09 record in `/data` carries `provenance` and `verification` fields.
- **The assessment instrument has not been validated.** There is no inter-rater reliability study;
  two people scoring the same organisation may differ, and by how much is unmeasured.
- **Control-to-case mappings are design hypotheses, not proven interceptions.** That a control
  addresses a failure mode observed in a case is not a claim that it would have prevented that
  case. See `SCOPE_AND_LIMITATIONS.md`.
