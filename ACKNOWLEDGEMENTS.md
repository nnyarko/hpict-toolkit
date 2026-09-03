# Acknowledgements

HPI-CT is built entirely from public information, using public tools. This page credits the sources and the tooling that made it possible. Crediting a source here does not imply that the source endorses, has reviewed, or is affiliated with this project.

## Data sources (all public)

The coded case base and the control evidence draw on public records from:

- **U.S. Department of Justice** — enforcement announcements, takedown releases, and charging documents (justice.gov).
- **Free Law Project / CourtListener** — federal and state court opinions and dockets (courtlistener.com). CourtListener is operated by the non-profit Free Law Project; its data is used here with thanks.
- **HHS Office of Inspector General (OIG)** — the List of Excluded Individuals/Entities (LEIE), oversight reports, and compliance guidance (oig.hhs.gov).
- **U.S. Government Accountability Office (GAO)** — oversight findings and the Green Book internal-control standard (gao.gov).
- **Centers for Medicare & Medicaid Services (CMS)** — the NPPES/NPI Registry, NCCI edits, Open Payments, PECOS enrollment files, and improper-payment measurement (cms.gov).
- **State enforcement authorities** — state Attorneys General and Medicaid Fraud Control Units, via their public announcements.
- **Additional registries referenced by the control library** — SSA (Death Master File), GSA (SAM.gov), HHS ACF (PARIS), state Secretaries of State, state corrections and Medicaid agencies, and state PDMP and EVV systems. The full registry, with custodians and access terms, is in [`/data/data_sources.csv`](data/data_sources.csv).

Works of the U.S. Government are not subject to copyright (17 U.S.C. § 105) and are used freely. What is original here is the coding and analysis, not the underlying records. See [`THIRD_PARTY_RIGHTS.md`](THIRD_PARTY_RIGHTS.md).

## Research tooling

- **Program Integrity Alliance (PIA)** — the PIA Connect / Integrity Lab tools were used as a discovery and harvest layer to locate the public DOJ, OIG, and GAO enforcement and oversight records that feed the case base. The coded facts come from those primary public records; no PIA-held proprietary dataset is redistributed here. With thanks to PIA for making public-integrity research more accessible.

## Standards

- **GAO Green Book** (Standards for Internal Control in the Federal Government) — public domain; the framework's five-component / seventeen-principle structure is sourced here.
- **COSO** Internal Control–Integrated Framework and the **COSO/ACFE** Fraud Risk Management Guide — copyrighted works of their owners, referenced by number and described in original wording, never reproduced. See [`THIRD_PARTY_RIGHTS.md`](THIRD_PARTY_RIGHTS.md).

"COSO" and "ACFE" are trademarks of their owners, used descriptively to identify the standards mapped to. No endorsement or affiliation is claimed.

## Software

- **Chart.js** and any other vendored open-source libraries, under their own licences (preserved in the vendored files).

## And

Everyone who has tried the toolkit and sent a correction or a suggestion. Corrections are logged in [`CHANGELOG.md`](CHANGELOG.md).

---

*Personal-capacity, public-source project. Acknowledgement is not endorsement. Last updated 2026-09.*
