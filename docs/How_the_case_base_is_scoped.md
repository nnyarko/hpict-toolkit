# How the case base is scoped

*Figures are from the case base as re-screened in October 2026 under coding rulebook v1.2. The screening was done by AI agents on one model, working from the official Justice Department text, and checked internally; it is not independent review, and the toolkit's authors are not lawyers.*

## What a row is

One row is one enforcement matter: a defendant or group of defendants, one scheme, and an official Justice Department source for it. Everything in the case base is an allegation or a resolution as the Department described it; defendants are presumed innocent unless a court has found otherwise. The published workbook carries no names, links or exact dollar figures.

## The four tests

A matter is counted only if all four hold.

1. **Health care.** It concerns fraud, waste or abuse in delivering or paying for health care.
2. **A payer.** The source alleges a false or improper claim on, or payment obtained from, a health care programme or payer: Medicare, Medicaid, TRICARE, veterans' or federal-employee health benefits, another government programme, or an insurer billed for health care. A kickback counts where the referrals it bought were billed to a payer.
3. **One matter, its own source.** The citation is that matter's own release or filing. An announcement of a takedown, a round-up of separate actions, or a task force never stands as a row.
4. **Source and date.** Justice Department releases or filings, 2023 or later.

The toolkit's controls protect payments. A matter with no claim on a payer cannot show a payment control failing, so it is not counted. Those matters are kept as an uncounted watch list because they point to gaps outside payment.

## Why the count changed

Versions 1.0.0 to 1.0.2 reported 1,586 coded cases. That was the count on 18 August 2026, when version 1.0.0 was released. The automated harvest went on adding rows after that count was taken, and by the time of the re-screening in October 2026 the case base held 1,605 rows, a net 19 more. Those later rows were never part of a published count, but they went through the same re-screening, so the table below accounts for 1,605 rows, not 1,586.

The earlier versions never stated the four tests above, and many rows had entered on a keyword match. All 1,605 rows were put through the re-screening. 1,599 were read against the official text. For the other 6 the official page could not be matched or could not be read; they are marked not read yet and are not counted. The re-screening, and the trace and spot checks behind it, are recorded in the project's working papers, which the maintainer holds and which are not part of this release.

| What happened to the rows | Rows |
|---|---:|
| Counted | 877 |
| Merged into another row of the same matter (a plea and a later sentencing, co-defendants of one scheme, or the same release on two pages) | 239 |
| Controlled-substance prescribing or diversion with no claim on a payer | 151 |
| Pandemic relief fraud (business loans, unemployment benefits) | 97 |
| An announcement or round-up, not one matter | 78 |
| Not health care | 63 |
| Borderline, unresolved, on the watch list | 39 |
| No claim on a payer, other | 30 |
| Theft or embezzlement from a provider | 13 |
| Tax, wage or consumer-protection matter | 8 |
| Not read yet (pending) | 6 |
| Not a Justice Department source | 3 |
| Manufacturer coupon programme | 1 |
| **All rows** (1,599 read, 6 not read yet) | **1,605** |

Rows are never deleted. Each carries its reason, so the history is reproducible. Of the rows counted, 555 carry a plain dollar figure that is summed (two more carry a figure followed by text in the same cell; those two are not summed).

## What did not change

The questionnaire and scoring are unchanged by the re-screening (control citations were corrected separately; see the changelog). Only the evidence behind the controls shrank: in every scheme category, fewer matters stand behind each control.

## By category

| Category | Rows before | Counted |
|---|---:|---:|
| Opioid / controlled substance / pharmacy | 253 | 66 |
| Telemedicine / genetic testing | 158 | 81 |
| Hospital/physician kickback (Stark/AKS) | 154 | 108 |
| Other health care fraud | 120 | 57 |
| COVID-19 testing | 117 | 21 |
| DME | 110 | 82 |
| Hospice / home health | 102 | 66 |
| Behavioral health / sober homes | 79 | 51 |
| Lab / urine drug testing | 78 | 63 |
| Compounding pharmacy | 68 | 48 |
| MA risk adjustment | 53 | 42 |
| Personal care / Medicaid | 53 | 34 |
| Provider enrollment / nominee / TCO | 52 | 26 |
| Physical therapy / chiropractic | 38 | 33 |
| Ambulance / transportation | 30 | 16 |
| Nursing home / substandard care | 26 | 10 |
| Wound care / amniotic allograft | 24 | 10 |
| Upcoding / E&M | 23 | 20 |
| FCA Stark/AKS settlement | 22 | 11 |
| Other / FCA | 17 | 15 |
| Pharma pricing / rebate (AKS) | 10 | 7 |
| Mental health / psychotherapy | 7 | 5 |
| Dental / pediatric | 6 | 3 |
| Autism/ABA therapy | 2 | 1 |
| Telehealth stimulant / Adderall prescribing | 1 | 1 |
| DME/catheters; Medicaid managed care eligibility | 2 | 0 |

The toolkit's published categories merge "FCA Stark/AKS settlement" into hospital/physician kickback (108 + 11 = 119 counted) and "Other / FCA" into other general health care fraud. The largest falls are in COVID-19 testing (relief fraud and announcements were swept in by keyword) and in opioid and controlled-substance matters (most had no claim on a payer).

## Limits

- The re-screening was done by AI agents; a second AI reader re-checked a sample, and the controlled-substance prescribing rows proposed for removal were read again from the full release text by AI. It is internal checking, not independent review; no person outside the project has checked it.
- Where the official page could not be read or matched, the row is marked pending and is not counted.
- Counts, not the dollar figures, are the reliable part. Dollar totals sum only the rows that state a headline figure, and understate the size of every category.
- A rule that counts only matters with a claim on a payer leaves out fraud that harmed the programme without one. The watch list is where that is kept.

## First re-read result

A first quarterly re-read of 50 counted rows (random draw, read from the official text, by an AI reader) found no row that failed the four tests, but 9 of 50 had a scheme category that clearly does not fit its definition and 8 more were arguable. Categories, stages and components were not re-coded in this release, so per-category counts are approximate and the total is the more reliable figure. The record of that re-read is in the project's working papers.

## How it is kept honest

The rulebook (v1.2) requires every new row to be screened against the four tests before it is coded, and a quarterly re-read of 50 rows from the official text, recording the share that fails. The first re-read was done on 27 October 2026.
