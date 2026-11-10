# Healthcare Claims Data Engineering & Analytics Platform

An end-to-end portfolio project demonstrating how synthetic healthcare
payer data can be ingested, validated, transformed, stored, reconciled,
and prepared for analytics.

> **Portfolio scope:** This repository is a technical demonstration
> built with synthetic data. It does not contain real patient
> information and does not represent production work performed for a
> healthcare payer.

## Project objectives

The platform is designed to demonstrate practical Data Engineer II
capabilities:

-   Ingest multiple source datasets through a repeatable batch process.
-   Preserve raw source records and ingestion metadata.
-   Validate required fields, data types, code formats, and referential
    integrity.
-   Separate accepted records from rejected records for investigation.
-   Transform validated data into analytics-ready structures.
-   Store curated data in a relational SQL Server warehouse.
-   Reconcile source and target record counts and financial totals.
-   Capture batch status, execution metrics, and data-quality results.
-   Publish claims and diagnostic-utilization analytics for business
    users.

## Architecture

The intended data flow is:

1.  **Source data:** synthetic medical claims, member eligibility,
    provider directory, and diagnosis/procedure reference data.
2.  **Ingestion:** Python batch ingestion, with AWS S3 used as the
    landing-storage target in the cloud-oriented design.
3.  **Bronze:** source-preserving raw records plus batch and source
    metadata.
4.  **Silver:** standardized, deduplicated, validated records; invalid
    records are written to a quarantine area with rejection reasons.
5.  **Gold:** business-ready facts, dimensions, aggregates, and
    analytical views.
6.  **Warehouse:** SQL Server relational tables and views for governed
    analytical access.
7.  **Consumption:** Power BI reports for claims trends, diagnostic
    utilization, provider performance, and data quality.

Python and PySpark support ingestion and transformation. SQL is used for
warehouse structures, constraints, reconciliation, and analytical views.
Operational controls include batch auditing, logging, monitoring, and
documentation.

See [the architecture and design overview](docs/01_project_overview.md)
for the detailed scope and design assumptions.

## Repository status

This repository begins with the project specification. Components will
be marked **Implemented**, **In progress**, or **Planned** as they are
developed and tested. The architecture diagram is a target-state design;
it must not be interpreted as proof that every service has already been
deployed.

## Proposed technology stack

  ------------------------------------------------------------------------
  Area                    Technology              Intended use
  ----------------------- ----------------------- ------------------------
  Programming             Python                  Ingestion, validation,
                                                  orchestration, tests

  Distributed processing  PySpark                 Scalable transformation
                                                  patterns

  Object storage          AWS S3                  Landing and file-based
                                                  data layers

  Relational warehouse    Microsoft SQL Server    Relational model,
                                                  constraints, indexes,
                                                  views

  Query language          T-SQL                   Warehouse
                                                  transformations and
                                                  reconciliation

  Analytics               Power BI                Claims and
                                                  diagnostic-utilization
                                                  reporting

  Version control         Git and GitHub          Source control,
                                                  reviewable changes,
                                                  project history
  ------------------------------------------------------------------------

The implementation will prioritize a runnable local development path.
Cloud deployment and Power BI connectivity will be documented separately
if they require credentials, paid services, or environment-specific
setup.

## Data privacy and safety

-   All data is synthetic and generated for this project.
-   No protected health information (PHI), real member identifiers, or
    real patient records are used.
-   Synthetic identifiers are not intended to identify actual people or
    providers.
-   Any future external sample data must be reviewed for licensing and
    privacy before inclusion.

## Planned repository structure

``` text
Healthcare-Claims-Data-Engineering/
├── README.md
├── requirements.txt
├── .gitignore
├── docs/
│   ├── 01_project_overview.md
│   ├── 02_architecture.md
│   ├── 03_data_model.md
│   ├── 04_data_dictionary.md
│   ├── 05_data_quality_framework.md
│   ├── 06_etl_design.md
│   ├── 07_data_lineage.md
│   └── 08_operational_runbook.md
├── data/
│   └── synthetic/
├── sql/
├── src/
│   ├── ingestion/
│   ├── transformations/
│   ├── quality/
│   ├── pipelines/
│   └── monitoring/
├── tests/
├── notebooks/
└── dashboards/
```

## Development roadmap

1.  Define the synthetic source schemas, business rules, and data
    dictionary.
2.  Generate reproducible synthetic claims, eligibility, provider,
    diagnosis, and procedure data.
3.  Design the SQL Server warehouse, including fact and dimension
    tables.
4.  Implement Python ingestion, validation, quarantine, and batch audit.
5.  Add PySpark transformations and curated data-layer outputs.
6.  Add automated tests, reconciliation checks, logging, and monitoring.
7.  Build analytical SQL views and Power BI report specifications.
8.  Document setup, execution, data lineage, limitations, and results.

## Important implementation principles

-   **Idempotency:** rerunning a batch should not create unintended
    duplicate target records.
-   **Traceability:** every processed record should be traceable to its
    source and load batch where feasible.
-   **Data quality:** validation failures should be measurable and
    explainable, not silently discarded.
-   **Reconciliation:** source-to-target counts and financial totals
    should be compared using documented rules.
-   **Security:** credentials belong in environment variables or a local
    secret store, never in source control.
-   **Honest scope:** tools and standards are described as implemented
    only after working code and evidence are added.

## Disclaimer

This is an educational portfolio project - I just want to show how great I am. It is not a clinical
decision-support system, a claims adjudication engine, or a substitute
for payer-specific policies, regulatory review, or production security
controls - I just want to show how useful I can be.
