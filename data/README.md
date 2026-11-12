# HPI-CT Open Data (`/data`)

Machine-readable exports, licensed **CC BY 4.0** (attribution required). One exception: SQL is code, so the SQL in `analytics_tests.json` (and in `analytics/*.md`) is under the **MIT** license ([`LICENSE`](../LICENSE)). All files are derived, de-identified, or public-domain-sourced. Reuse freely.

| File | What it is | Records |
|---|---|---|
| `controls.json` / `.csv` | The full control library, each control with module, type (preventive/detective/corrective), operational stage, Green Book principle, governing citation, analytics technique, applies-to, trigger, owner, evidence artifact, failure mode, REG/DS links. | 173 |
| `questionnaire.json` | The evidence-anchored self-assessment: questions by module, with tiers, weights, and scored options. | 247 |
| `rules.json` / `.csv` | Regulatory rules dictionary, REG-IDs with citation and the controls that cite them. | 88 |
| `data_sources.json` / `.csv` | Data-source registry (LEIE, SAM, DMF, PARIS, NPPES, NCCI, etc.) with access tier, cadence, match keys, consuming controls. | 20 |
| `diagnosis_matrix.json` / `.csv` | Control-failure diagnosis matrix, detection-flag families → implied scheme → stage → the HPI-CT controls that address it. | 29 |
| `cases.json` | **De-identified** case exemplars (DOJ + state), scheme, stage, failed control, and COSO/Green Book mapping. Party names, case names, URLs, and exact dates removed. Row-level dollar amounts are banded (`amt_band`). As the file stands, each DOJ category also carries a total (`billed`) in $ millions to one decimal, summed over the counted rows in that category that state a figure; that total is not banded. The `ctrl` field uses the case base's own shorthand (C-01 to C-10) for the 21 DOJ categories, not library control IDs; only the 9 state patterns name library controls. | 21 DOJ categories + 9 state patterns |
| `public_stats.json` | Headline counts used by the toolkit (controls, questions, coded_cases 877). | n/a |
| `analytics_tests.json` | 128 reference detection patterns in 8 families, each with hypothesis, check, SQL, threshold guidance, the controls it supports and the failure mode a hit points to. Reference patterns, run on invented data only. | 128 |
| `improper_payment_weighting.json` / `.csv` | Improper-payment weighting reference. | n/a |

## Not included (held by the maintainer)
The **row-level identified coded dataset and raw intake** are not published here. `cases.json` is the de-identified analytical subset. The identified source crosswalk is available to researchers on request under a short data-use understanding, nicholasnyarko@gmail.com.

## Attribution
> HPI-CT — Healthcare Program Integrity Control Toolkit, created by Nicholas Nyarko, licensed under CC BY 4.0.

*All case references are allegations from public sources; defendants presumed innocent. Confirm regulatory citations against primary `.gov` sources.*
