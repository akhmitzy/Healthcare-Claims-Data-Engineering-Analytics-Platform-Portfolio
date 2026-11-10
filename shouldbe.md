# Healthcare Claims Data Platform --- Project Blueprint

## 1. The architecture

### Healthcare claims data platform

``` text
SOURCE SYSTEMS
Medical claims · Member eligibility · Provider directory
Diagnosis/procedure reference data
        |
        v
1. INGESTION — BRONZE
Python / PySpark · CSV/JSON · S3 landing zone · Batch metadata
Preserve source data
        |
        v
2. VALIDATION & TRANSFORMATION — SILVER
SQL / Python · Standardization · Deduplication
Referential integrity · Data quality rules
Reject or quarantine invalid records
        |
        v
3. DATA WAREHOUSE — GOLD
SQL Server · Fact and dimension tables
Aggregations · Reconciliation
        |
        v
4. CONSUMPTION
Power BI / Tableau · Claims analytics · Provider analytics
Diagnostic utilization · Data science
```

This architecture maps directly to the job. It also lets you demonstrate
experience with AWS/S3 and Databricks in a way that complements Avalon's
Microsoft SQL Server environment.

## 2. What the project should contain

  -----------------------------------------------------------------------
  Component                           What you demonstrate
  ----------------------------------- -----------------------------------
  Synthetic source data               Medical claims, members, providers,
                                      diagnosis and procedure codes

  ETL pipeline                        Extract, validate, transform, load

  SQL Server warehouse                Fact and dimension tables

  Data quality framework              Missing values, duplicates, invalid
                                      codes, impossible dates, amount
                                      discrepancies

  Reconciliation                      Source-to-target record counts and
                                      financial totals

  Monitoring                          Run IDs, execution status, rejected
                                      records, error logs

  Analytics                           Claims cost, utilization, provider
                                      patterns, diagnostic testing

  Documentation                       Data dictionary, architecture,
                                      lineage, operational instructions

  GitHub                              Version control, project structure,
                                      tests, reproducibility
  -----------------------------------------------------------------------

### Healthcare data model

The central fact table should be `FactClaimLine`, with one row per claim
service line.

**Dimensions**

-   `DimMember`
-   `DimProvider`
-   `DimDate`
-   `DimDiagnosis`
-   `DimProcedure`
-   `DimPayer`
-   `DimPlaceOfService`

**Measures**

-   Billed amount
-   Allowed amount
-   Paid amount
-   Member responsibility
-   Units of service
-   Claim line count

A claim may contain multiple service lines, so the data model must avoid
counting the same claim multiple times when calculating claim-level
metrics. For example, use a distinct count of `ClaimID` for claim counts
rather than counting rows in `FactClaimLine`.

## 3. Code sample: SQL Server warehouse

This is the kind of SQL to put in the repository. It demonstrates a
relational model rather than a collection of simple `SELECT` statements.

**File:** `sql/01_create_warehouse.sql`

``` sql
CREATE SCHEMA warehouse;
GO

CREATE TABLE warehouse.DimMember
(
    MemberKey       INT IDENTITY(1,1) PRIMARY KEY,
    MemberID        VARCHAR(50) NOT NULL,
    EffectiveDate   DATE NOT NULL,
    EndDate         DATE NULL,
    IsCurrent       BIT NOT NULL DEFAULT 1,
    GenderCode      VARCHAR(10) NULL,
    BirthYear       SMALLINT NULL,

    CONSTRAINT UQ_DimMember
        UNIQUE (MemberID, EffectiveDate)
);
GO

CREATE TABLE warehouse.DimProvider
(
    ProviderKey     INT IDENTITY(1,1) PRIMARY KEY,
    ProviderID      VARCHAR(50) NOT NULL,
    ProviderType    VARCHAR(50) NULL,
    SpecialtyCode   VARCHAR(30) NULL,
    EffectiveDate   DATE NOT NULL,
    EndDate         DATE NULL,
    IsCurrent       BIT NOT NULL DEFAULT 1
);
GO

CREATE TABLE warehouse.FactClaimLine
(
    ClaimLineKey       BIGINT IDENTITY(1,1) PRIMARY KEY,
    ClaimID            VARCHAR(50) NOT NULL,
    ClaimLineNumber    INT NOT NULL,

    MemberKey          INT NOT NULL,
    ProviderKey        INT NOT NULL,
    ServiceDateKey     INT NOT NULL,

    ProcedureCode      VARCHAR(20) NULL,
    DiagnosisCode      VARCHAR(20) NULL,

    BilledAmount       DECIMAL(18,2) NOT NULL,
    AllowedAmount      DECIMAL(18,2) NOT NULL,
    PaidAmount         DECIMAL(18,2) NOT NULL,

    LoadBatchID        UNIQUEIDENTIFIER NOT NULL,
    LoadTimestamp      DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME(),

    CONSTRAINT UQ_ClaimLine
        UNIQUE (ClaimID, ClaimLineNumber),

    CONSTRAINT FK_Claim_Member
        FOREIGN KEY (MemberKey)
        REFERENCES warehouse.DimMember(MemberKey),

    CONSTRAINT FK_Claim_Provider
        FOREIGN KEY (ProviderKey)
        REFERENCES warehouse.DimProvider(ProviderKey),

    CONSTRAINT CK_Claim_Amounts
        CHECK (
            BilledAmount >= 0
            AND AllowedAmount >= 0
            AND PaidAmount >= 0
        )
);
GO

CREATE INDEX IX_FactClaimLine_ServiceDate
ON warehouse.FactClaimLine(ServiceDateKey);

CREATE INDEX IX_FactClaimLine_Provider
ON warehouse.FactClaimLine(ProviderKey);
GO
```

This provides an opportunity to explain primary keys, foreign keys,
uniqueness, constraints, indexing, and the grain of a fact table during
an interview.

> **Implementation note:** The example includes member and provider
> dimensions, but a production-ready version should also create and
> populate the date, diagnosis, procedure, payer, and place-of-service
> dimensions. `ServiceDateKey` should reference `DimDate` through a
> foreign key once that table is created.

## 4. Code sample: data quality validation in Python

The data quality framework is particularly important for Avalon.
Healthcare analytics is only useful when the underlying data is
reliable.

**File:** `src/quality/claim_validation.py`

``` python
from dataclasses import dataclass

import pandas as pd


@dataclass
class ValidationResult:
    valid_records: pd.DataFrame
    rejected_records: pd.DataFrame
    quality_summary: dict


def validate_claims(df: pd.DataFrame) -> ValidationResult:
    required = [
        "claim_id",
        "claim_line_number",
        "member_id",
        "provider_id",
        "service_date",
        "billed_amount",
        "allowed_amount",
        "paid_amount",
    ]

    missing_columns = set(required) - set(df.columns)
    if missing_columns:
        raise ValueError(
            f"Missing required columns: {sorted(missing_columns)}"
        )

    data = df.copy()

    # Normalize dates and numeric fields.
    data["service_date"] = pd.to_datetime(
        data["service_date"], errors="coerce"
    )

    amount_columns = [
        "billed_amount",
        "allowed_amount",
        "paid_amount",
    ]
    for column in amount_columns:
        data[column] = pd.to_numeric(data[column], errors="coerce")

    # Build row-level validation flags.
    checks = pd.DataFrame(index=data.index)
    checks["missing_identifiers"] = data[
        ["claim_id", "member_id", "provider_id"]
    ].isna().any(axis=1)
    checks["invalid_service_date"] = data["service_date"].isna()
    checks["invalid_amounts"] = data[amount_columns].isna().any(axis=1)
    checks["negative_amounts"] = (data[amount_columns] < 0).any(axis=1)
    checks["duplicate_claim_line"] = data.duplicated(
        ["claim_id", "claim_line_number"], keep="first"
    )

    # These financial relationships are illustrative business rules.
    # Confirm the applicable rules with the payer before using them.
    checks["suspicious_amount_relationship"] = (
        (data["allowed_amount"] > data["billed_amount"])
        | (data["paid_amount"] > data["allowed_amount"])
    )

    rejected_mask = checks.any(axis=1)

    valid_records = data.loc[~rejected_mask].copy()
    rejected_records = data.loc[rejected_mask].copy()

    # Preserve rejection reasons so rejected rows can be investigated.
    rejected_records["rejection_reasons"] = checks.loc[
        rejected_mask
    ].apply(
        lambda row: ", ".join(row.index[row].tolist()),
        axis=1,
    )

    quality_summary = {
        "input_records": int(len(data)),
        "valid_records": int(len(valid_records)),
        "rejected_records": int(len(rejected_records)),
        "valid_rate": (
            float(len(valid_records) / len(data))
            if len(data)
            else 0.0
        ),
        "rejection_counts_by_rule": {
            rule: int(checks[rule].sum())
            for rule in checks.columns
        },
    }

    return ValidationResult(
        valid_records=valid_records,
        rejected_records=rejected_records,
        quality_summary=quality_summary,
    )
```

**Why this is valuable:** It demonstrates explicit validation rules,
rejected-record handling, reproducibility, and measurable data quality.
In a production implementation, add rule versioning, configurable
thresholds, structured logs, and tests for edge cases.

The financial checks are illustrative business rules and must be agreed
with the payer's claims policy; they are not universal rules for every
claim. The example also treats repeated claim ID/line-number
combinations as duplicates; production logic should account for the
source system's adjustment, replacement, and reversal conventions.

## 5. Code sample: SQL data quality and reconciliation

**File:** `sql/02_data_quality.sql`

This script checks whether warehouse data is internally consistent.

``` sql
-- 1. Detect duplicate claim service lines
SELECT
    ClaimID,
    ClaimLineNumber,
    COUNT(*) AS DuplicateCount
FROM warehouse.FactClaimLine
GROUP BY
    ClaimID,
    ClaimLineNumber
HAVING COUNT(*) > 1;


-- 2. Identify suspicious financial relationships
SELECT
    ClaimID,
    ClaimLineNumber,
    BilledAmount,
    AllowedAmount,
    PaidAmount
FROM warehouse.FactClaimLine
WHERE
    AllowedAmount > BilledAmount
    OR PaidAmount > AllowedAmount;


-- 3. Reconcile financial totals by batch
SELECT
    LoadBatchID,
    COUNT(*) AS LoadedLines,
    SUM(BilledAmount) AS TotalBilled,
    SUM(AllowedAmount) AS TotalAllowed,
    SUM(PaidAmount) AS TotalPaid
FROM warehouse.FactClaimLine
GROUP BY LoadBatchID;
```

For a stronger project, implement a reconciliation table named
`audit.ETLBatchAudit`.

  Column            Purpose
  ----------------- -----------------------------
  `BatchID`         Unique execution identifier
  `SourceSystem`    Origin of the data
  `StartTime`       Pipeline start
  `EndTime`         Pipeline completion
  `SourceCount`     Records received
  `LoadedCount`     Records accepted
  `RejectedCount`   Records quarantined
  `Status`          Success / Failed / Partial
  `ErrorMessage`    Failure details

This is particularly relevant to the requirement to monitor, maintain,
and improve production pipelines.

## 6. Python pipeline orchestration

**File:** `src/pipelines/run_claims_pipeline.py`

The following is a simplified orchestration example. It connects
ingestion, validation, and audit reporting. It writes accepted and
rejected records to separate Parquet files; a production version would
also load the accepted records into SQL Server and persist the audit
record.

``` python
from datetime import datetime, timezone
from pathlib import Path
from uuid import uuid4

import pandas as pd

from quality.claim_validation import validate_claims


def run_pipeline(source_file: str, output_directory: str) -> dict:
    batch_id = str(uuid4())
    started_at = datetime.now(timezone.utc)

    output_path = Path(output_directory)
    output_path.mkdir(parents=True, exist_ok=True)

    print(f"Starting claims pipeline: {batch_id}")

    try:
        # Extract
        raw = pd.read_csv(source_file)

        # Validate and transform
        result = validate_claims(raw)

        # Persist accepted and rejected data.
        # Include the batch ID in output names to avoid overwriting
        # the results of earlier pipeline runs.
        result.valid_records.to_parquet(
            output_path / f"valid_claims_{batch_id}.parquet",
            index=False,
        )
        result.rejected_records.to_parquet(
            output_path / f"rejected_claims_{batch_id}.parquet",
            index=False,
        )

        completed_at = datetime.now(timezone.utc)

        audit = {
            "batch_id": batch_id,
            "started_at": started_at.isoformat(),
            "completed_at": completed_at.isoformat(),
            "source_count": result.quality_summary["input_records"],
            "loaded_count": result.quality_summary["valid_records"],
            "rejected_count": result.quality_summary["rejected_records"],
            "status": "Success",
            "error_message": None,
            "quality_summary": result.quality_summary,
        }

        print(f"Pipeline completed: {batch_id}")
        print(result.quality_summary)
        return audit

    except Exception as exc:
        completed_at = datetime.now(timezone.utc)

        audit = {
            "batch_id": batch_id,
            "started_at": started_at.isoformat(),
            "completed_at": completed_at.isoformat(),
            "source_count": None,
            "loaded_count": 0,
            "rejected_count": 0,
            "status": "Failed",
            "error_message": str(exc),
        }

        print(f"Pipeline failed: {batch_id}: {exc}")
        return audit


if __name__ == "__main__":
    run_pipeline(
        source_file="data/raw/claims.csv",
        output_directory="data/processed",
    )
```

For the finished version, add:

-   SQL Server loading through SQLAlchemy/ODBC.
-   Incremental loads and idempotent processing.
-   Retry logic and failure handling.
-   Structured logging.
-   Pipeline configuration outside source code.
-   Unit tests with `pytest`.
-   A scheduler, such as SQL Server Agent or an orchestration tool.

I would also include a PySpark implementation of the transformation
layer, since that connects the project to existing Spark/Databricks
experience.

## 7. Analytics that connect directly to Avalon

The platform should not stop at loading claims. It should produce useful
analytical data products.

### A. Diagnostic testing utilization

-   Number of diagnostic tests by procedure code.
-   Tests per 1,000 members.
-   Utilization by provider specialty.
-   Changes in utilization over time.
-   Geographic differences in testing patterns.

### B. Claims financial analytics

-   Billed versus allowed versus paid amounts.
-   Average cost per service.
-   Cost by provider and specialty.
-   High-cost procedure categories.
-   Monthly trends.

### C. Data quality dashboard

-   Percentage of valid claims.
-   Duplicate claim-line rate.
-   Missing provider/member identifiers.
-   Invalid dates and codes.
-   Rejected records by source system.
-   Pipeline execution success rate.

### Dashboard design references

The uploaded dashboard image and the following examples can be used as
visual inspiration for the Power BI/Tableau deliverables. Use synthetic
data and label the dashboards clearly as demonstrations---not actual
Avalon data or clinical recommendations.

-   [One Framework, All Calendars: DAX MTD/QTD/YTD for Calendar and
    Fiscal Years in Power BI](https://medium.com/)
-   [From Raw Claims Data to Insights: Building a PCP Utilization
    Dashboard in Power BI](https://medium.com/)
-   [What is Data Quality?](https://www.ibm.com/topics/data-quality)

------------------------------------------------------------------------

## Suggested GitHub repository structure

``` text
healthcare-claims-data-platform/
├── data/
│   ├── raw/
│   ├── reference/
│   └── processed/
├── docs/
│   ├── architecture.md
│   ├── data_dictionary.md
│   └── data_lineage.md
├── sql/
│   ├── 01_create_warehouse.sql
│   ├── 02_data_quality.sql
│   └── 03_reconciliation.sql
├── src/
│   ├── quality/
│   │   └── claim_validation.py
│   └── pipelines/
│       └── run_claims_pipeline.py
├── tests/
│   └── test_claim_validation.py
├── dashboards/
├── requirements.txt
└── README.md
```

## Project positioning

Present this as a synthetic healthcare claims data engineering and
analytics project. It demonstrates an end-to-end workflow: source
ingestion, data validation, transformation, relational warehousing,
reconciliation, monitoring, and business-facing analytics.

The architecture deliberately combines Python/PySpark, AWS/S3, and
Databricks concepts with a SQL Server warehouse and Power BI/Tableau
consumption layer. This shows how cloud and Spark experience can
complement a Microsoft SQL Server-centered environment without claiming
that the demonstration uses Avalon's actual systems or data.
