# HPI-CT Open Data (`/data`)

Machine-readable exports, licensed **CC BY 4.0** (attribution required). All files are derived, de-identified, or public-domain-sourced. Reuse freely.

| File | What it is | Records |
|---|---|---|
| `controls.json` / `.csv` | The full control library — each control with module, type (preventive/detective/corrective), operational stage, Green Book principle, governing citation, analytics technique, applies-to, trigger, owner, evidence artifact, failure mode, REG/DS links. | 160 |
| `questionnaire.json` | The evidence-anchored self-assessment — questions by module, with tiers, weights, and scored options. | 224 |
| `rules.json` / `.csv` | Regulatory rules dictionary — REG-IDs with citation and the controls that cite them. | 78 |
| `data_sources.json` / `.csv` | Data-source registry (LEIE, SAM, DMF, PARIS, NPPES, NCCI, etc.) with access tier, cadence, match keys, consuming controls. | 18 |
| `diagnosis_matrix.json` / `.csv` | Control-failure diagnosis matrix — detection-flag families → implied scheme → stage → the HPI-CT controls that address it. | 29 |
| `cases.json` | **De-identified** case exemplars (DOJ + state) — scheme, stage, failed control, and COSO/Green Book mapping. Party names, case names, URLs, and exact dates removed; dollar figures banded. | 22 DOJ + 9 state |
| `public_stats.json` | Headline counts used by the toolkit. | — |
| `improper_payment_weighting.json` / `.csv` | Improper-payment weighting reference. | — |

## Not included (held by the maintainer)
The **row-level identified coded dataset and raw intake** are not published here. `cases.json` is the de-identified analytical subset. The identified source crosswalk is available to researchers on request under a short data-use understanding — focusfusionuniverse@gmail.com.

## Attribution
> HPI-CT — Healthcare Program Integrity Control Toolkit, created by Nicholas Nyarko, licensed under CC BY 4.0.

*All case references are allegations from public sources; defendants presumed innocent. Confirm regulatory citations against primary `.gov` sources.*
