# Healthcare Data Pipeline and Analytics Using SQL

**A SQL Project: Health Care Data Pipeline and Analytics (2026)**

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Dataset Overview](#dataset-overview)
3. [Data Quality Assessment](#data-quality-assessment)
4. [Data Transformation](#data-transformation)
5. [Data Cleaning and Error Correction](#data-cleaning-and-error-correction)
6. [Feature Engineering](#feature-engineering)
7. [Data Modeling and Relational Schema](#data-modeling-and-relational-schema)
8. [Identify Revenue Leakage Pathways](#identify-revenue-leakage-pathways)
9. [Audit Clinical Wait Time SLA Breaches](#audit-clinical-wait-time-sla-breaches)
10. [Profile Demographic Care Vulnerabilities](#profile-demographic-care-vulnerabilities)
11. [Track Triage Operational Efficiency](#track-triage-operational-efficiency)
12. [Evaluate Insurance Collection Aging Metrics](#evaluate-insurance-collection-aging-metrics)
13. [Key Findings and Insights](#key-findings-and-insights)
14. [Recommendations](#recommendations)
15. [Conclusion](#conclusion)

---

## Project Overview

This project builds a complete healthcare analytics pipeline in Microsoft SQL Server, taking three raw operational tables from unreliable to analysis ready. It covers a full data quality audit, transformation, cleaning, feature engineering, and a proper relational schema with enforced keys. On top of that clean foundation, it answers five core business objectives spanning finance and clinical operations. Every step below is backed by the actual T dash SQL query used to solve it. The goal is a pipeline that is not just accurate once, but trustworthy and repeatable every time it runs.

**Tools used:** Microsoft SQL Server, T-SQL (Transact-SQL)

---

## Dataset Overview

The pipeline draws from three raw source tables, each covering a distinct part of hospital operations, patient identity, billing, and clinical activity.

| Table | What It Holds | Key Columns |
| --- | --- | --- |
| patient_registry (raw) | Master demographic profile for every patient in the system | patient_id, full_name, dob, gender, insurance_provider |
| billing_invoices (raw) | Every invoice issued to a patient, and whether it has been paid | invoice_id, patient_id, billing_date, amount_charged, payment_status |
| clinical_telemetry (raw) | Admission and triage timing data, vitals, and readmission flags | telemetry_id, patient_id, admission_timestamp, doctor_seen_timestamp, blood_pressure, readmitted |

None of these tables arrived analysis ready. Each one carried the kind of inconsistent formatting, duplication, and quiet gaps typical of real operational exports, which is exactly what the next phase was built to catch.

---

## Data Quality Assessment

Before any fix was applied, six categories of anomaly were audited and measured directly against the raw data. This phase exists to turn a vague sense that the data is messy into an exact, countable diagnosis. Nothing gets cleaned here, only identified.

### Anomaly One: Duplicate Records and Business Key Integrity

```sql
--DATA QUALITY AUDIT PHASE:
-- ANOMALY 1: DUPLICATE RECORDS & BUSINESS KEY INTEGRITY AUDIT
PRINT '--- ANOMALY 1: DUPLICATE RECORDS CHECK ---';

-- Check 1A: Compare total physical row counts against unique natural business keys
SELECT 
    'patient_registry' AS table_name, 
    COUNT(*) AS total_rows, 
    COUNT(DISTINCT patient_id) AS unique_business_keys 
FROM dbo.patient_registry
UNION ALL
SELECT 'billing_invoices', COUNT(*), COUNT(DISTINCT invoice_id) FROM dbo.billing_invoices
UNION ALL
SELECT 'clinical_telemetry', COUNT(*), COUNT(DISTINCT telemetry_id) FROM dbo.clinical_telemetry;

-- Check 1B: Isolate duplicated patient profiles in the master registry lookup
SELECT full_name, COUNT(*) as occurrences, COUNT(DISTINCT patient_id) as unique_ids
FROM dbo.patient_registry
GROUP BY full_name
HAVING COUNT(*) > 1
ORDER BY occurrences DESC;
GO
```

Check one A lines up total row counts against distinct business keys across all three tables at once, exposing duplication the instant those two numbers disagree. Check one B goes further, grouping by patient name to catch the same person registered twice under different identifiers.

### Anomaly Two and Four: Missing Values and Unstandardized Text

```sql
-- ANOMALY 2 & 4: MISSING VALUES (NULLS/BLANKS) & UNSTANDARDIZED TEXT AUDIT
PRINT '--- ANOMALY 2 & 4: MISSING VALUES & UNSTANDARDIZED TEXT ---';

-- Check 2A: Profile Patient Registry columns for explicit NULLs, empty gaps, or spaces
SELECT 
    SUM(CASE WHEN patient_id IS NULL OR LTRIM(RTRIM(patient_id)) = '' THEN 1 ELSE 0 END) as patient_id_anomalies,
    SUM(CASE WHEN full_name IS NULL OR LTRIM(RTRIM(full_name)) = '' THEN 1 ELSE 0 END) as full_name_anomalies,
    SUM(CASE WHEN dob IS NULL OR LTRIM(RTRIM(dob)) = '' THEN 1 ELSE 0 END) as dob_anomalies,
    SUM(CASE WHEN gender IS NULL OR LTRIM(RTRIM(gender)) = '' THEN 1 ELSE 0 END) as gender_anomalies,
    SUM(CASE WHEN insurance_provider IS NULL OR LTRIM(RTRIM(insurance_provider)) = '' THEN 1 ELSE 0 END) as insurance_anomalies
FROM dbo.patient_registry;

-- Check 2B: Audit text metrics for hidden leading/trailing spacing issues
SELECT patient_id, full_name, LEN(full_name) as length_raw, LEN(TRIM(full_name)) as length_trimmed
FROM dbo.patient_registry
WHERE LEN(full_name) != LEN(TRIM(full_name));

-- Check 2C: Isolate manual placeholder variants (UNKNOWN, N/A, blanks)
SELECT insurance_provider, COUNT(*) as occurrence_count
FROM dbo.patient_registry
WHERE insurance_provider IN ('UNKNOWN', 'N/A', 'Unknown', '', ' ')
GROUP BY insurance_provider;
GO
```

Check two A profiles every core column for true nulls and blank strings hiding as data. Check two B compares raw text length against trimmed length to catch invisible leading or trailing spaces. Check two C hunts down placeholder text like unknown or not applicable, sitting in a field that should either hold a real value or be genuinely blank.

### Anomaly Three and Six: Data Constraints and Negative Values

```sql
-- ANOMALY 3 & 6: GENERAL DATA QUALITY CONSTRAINTS & NEGATIVE VALUES AUDIT
PRINT '--- ANOMALY 3 & 6: DATA CONSTRAINTS & NEGATIVE VALUES ---';

-- Check 3A: Audit for contaminated date text strings that fail native calendar casting
SELECT dob, COUNT(*) as regional_format_occurrences
FROM dbo.patient_registry
WHERE TRY_CAST(dob AS DATE) IS NULL
GROUP BY dob;

-- Check 3B: Profile billing balances for non-numeric formatting text flags ($ or USD)
SELECT amount_charged, COUNT(*) as uncastable_currency_count
FROM dbo.billing_invoices
WHERE TRY_CAST(REPLACE(REPLACE(REPLACE(amount_charged, '$', ''), 'USD', ''), ' ', '') AS DECIMAL(10,2)) IS NULL
GROUP BY amount_charged;

-- Check 6A: Detect impossible negative financial billing parameters
SELECT invoice_id, patient_id, amount_charged 
FROM dbo.billing_invoices
WHERE amount_charged LIKE '-%' 
   OR TRY_CAST(REPLACE(REPLACE(REPLACE(amount_charged, '$', ''), 'USD', ''), ' ', '') AS DECIMAL(10,2)) < 0;
GO
```

Check three A attempts a native date cast on every date of birth value, grouping every failure by its raw text to expose regional format mismatches. Check three B does the same to billing amounts, catching currency symbols hiding inside a number field. Check six A flags any charge that is impossibly negative, a clear sign of a refund miscoded as a normal invoice.

### Anomaly Five: Inconsistent IDs and Referential Orphan Keys

```sql
-- ANOMALY 5: INCONSISTENT IDS & REFERENTIAL ORPHAN KEY AUDIT
PRINT '--- ANOMALY 5: REFERENTIAL INTEGRITY ORPHAN CHECKS ---';

-- Check 5A: Validate child billing table transactions mapping to non-existent lookup keys
SELECT DISTINCT b.patient_id AS orphaned_billing_patient_id
FROM dbo.billing_invoices b
WHERE b.patient_id NOT IN (SELECT DISTINCT patient_id FROM dbo.patient_registry WHERE patient_id IS NOT NULL);

-- Check 5B: Validate child telemetry metrics mapping to non-existent lookup keys
SELECT DISTINCT c.patient_id AS orphaned_telemetry_patient_id
FROM dbo.clinical_telemetry c
WHERE c.patient_id NOT IN (SELECT DISTINCT patient_id FROM dbo.patient_registry WHERE patient_id IS NOT NULL);
GO
```

Both checks confirm that every patient identifier referenced in billing or telemetry can be traced back to a real row in the master registry. Any identifier that fails is an orphan, a transaction or vital sign logged against a patient who technically does not exist in the system. By the end of this phase, every category of damage had a query and a row count attached to it, ready for the next stage to actually fix.

---

## Data Transformation

With every issue mapped, this phase moves records out of raw staging tables into their clean destination tables, applying the core fixes as data lands for the first time.

```sql
--DATA TRANSFORMATION PHASE:

DELETE FROM dbo.fact_clinical_telemetry;
DELETE FROM dbo.fact_billing_invoices;
DELETE FROM dbo.dim_patient_registry;
GO

INSERT INTO dbo.dim_patient_registry (
    patient_id, full_name, first_name, last_name, date_of_birth, gender, insurance_provider, record_created_date, record_updated_date
)
SELECT 
    patient_id, -- Removed TRIM wrapper to fix the BIT data type error
    TRIM(full_name),
    TRIM(LEFT(TRIM(full_name), CHARINDEX(' ', TRIM(full_name) + ' ') - 1)),
    TRIM(REVERSE(LEFT(REVERSE(TRIM(full_name)), CHARINDEX(' ', REVERSE(TRIM(full_name)) + ' ') - 1))),
    CASE 
        WHEN dob LIKE '%/%/%' AND CHARINDEX('/', dob) = 5 THEN CONVERT(DATE, dob, 111)
        WHEN dob LIKE '%/%/%' AND CHARINDEX('/', dob) <= 3 THEN CONVERT(DATE, dob, 101)
        WHEN dob LIKE '%-%-%' THEN CONVERT(DATE, dob, 105)
        ELSE NULL 
    END,
    CASE WHEN UPPER(LEFT(LTRIM(gender), 1)) = 'M' THEN 'M' WHEN UPPER(LEFT(LTRIM(gender), 1)) = 'F' THEN 'F' ELSE 'U' END,
    CASE WHEN UPPER(TRIM(ISNULL(insurance_provider, ''))) IN ('UNKNOWN', 'N/A', '') THEN 'Uninsured' ELSE UPPER(TRIM(insurance_provider)) END,
    GETDATE(),
    GETDATE()
FROM dbo.patient_registry_raw
WHERE patient_id IS NOT NULL;
GO

INSERT INTO dbo.fact_billing_invoices (
    invoice_id, patient_key, patient_id, billing_date, amount_charged, payment_status, record_created_date, record_updated_date
)
SELECT 
    UPPER(TRIM(b.invoice_id)),
    p.patient_key, 
    b.patient_id,
    CASE 
        WHEN b.billing_date LIKE '%/%/%' THEN CONVERT(DATE, b.billing_date, 103) 
        ELSE CONVERT(DATE, b.billing_date, 120) 
    END,
    TRY_CAST(LTRIM(RTRIM(REPLACE(REPLACE(REPLACE(b.amount_charged, '$', ''), 'USD', ''), ' ', ''))) AS DECIMAL(10,2)),
    CASE 
        WHEN LTRIM(RTRIM(b.payment_status)) = '' OR b.payment_status IS NULL THEN 'Pending'
        ELSE UPPER(LEFT(LTRIM(b.payment_status), 1)) + LOWER(SUBSTRING(LTRIM(b.payment_status), 2, LEN(b.payment_status)))
    END,
    GETDATE(),
    GETDATE()
FROM dbo.billing_invoices_raw b
INNER JOIN dbo.dim_patient_registry p ON b.patient_id = p.patient_id 
WHERE b.invoice_id IS NOT NULL;
GO

INSERT INTO dbo.fact_clinical_telemetry (
    telemetry_id, patient_key, patient_id, admission_timestamp, doctor_seen_timestamp, systolic_bp, diastolic_bp, readmitted, record_created_date, record_updated_date
)
SELECT 
    UPPER(TRIM(t.telemetry_id)),
    p.patient_key,
    t.patient_id,
    TRY_CAST(t.admission_timestamp AS DATETIME),
    TRY_CAST(t.doctor_seen_timestamp AS DATETIME),
    TRY_CAST(LEFT(t.blood_pressure, CHARINDEX('/', t.blood_pressure) - 1) AS INT),
    TRY_CAST(SUBSTRING(t.blood_pressure, CHARINDEX('/', t.blood_pressure) + 1, LEN(t.blood_pressure)) AS INT),
    CASE WHEN UPPER(TRIM(t.readmitted)) IN ('Y', 'YES') THEN 'Y' ELSE 'N' END,
    GETDATE(),
    GETDATE()
FROM dbo.clinical_telemetry_raw t
INNER JOIN dbo.dim_patient_registry p ON t.patient_id = p.patient_id 
WHERE t.telemetry_id IS NOT NULL 
  AND t.blood_pressure LIKE '%/%';
GO
```

The load clears destination tables first so re-runs never duplicate existing rows. The patient load splits full names into first and last name, normalizes gender to one letter, and folds every insurance placeholder into a single honest label, uninsured. Billing and clinical loads both join back to the fresh patient dimension, which quietly closes the orphaned key problem, while stripping currency symbols, splitting blood pressure readings, and standardizing dates along the way.

---

## Data Cleaning and Error Correction

Transformation moved the data into its new home. This pass tightens exactly how each value is standardized in place before any report gets built on top of it.

### Cleaning the Patient Registry Table

```sql
--DATA CLEANING PHASE:
--Clean patient registry table
-- Enforce clean string trimming, case overrides, proper case standardization, and regional date conversions
INSERT INTO dbo.dim_patient_registry (
    patient_id, 
    full_name, 
    first_name, 
    last_name, 
    date_of_birth, 
    gender, 
    insurance_provider, 
    record_created_date, 
    record_updated_date
)
SELECT 
    UPPER(TRIM(patient_id)) AS patient_id,
    
    -- Data Cleaning: Proper Case Full Name (Stitch the proper first and last names together)
    UPPER(LEFT(TRIM(LEFT(TRIM(full_name), CHARINDEX(' ', TRIM(full_name) + ' ') - 1)), 1)) + 
    LOWER(SUBSTRING(TRIM(LEFT(TRIM(full_name), CHARINDEX(' ', TRIM(full_name) + ' ') - 1)), 2, LEN(TRIM(LEFT(TRIM(full_name), CHARINDEX(' ', TRIM(full_name) + ' ') - 1))))) + 
    ' ' + 
    UPPER(LEFT(TRIM(REVERSE(LEFT(REVERSE(TRIM(full_name)), CHARINDEX(' ', REVERSE(TRIM(full_name)) + ' ') - 1))), 1)) + 
    LOWER(SUBSTRING(TRIM(REVERSE(LEFT(REVERSE(TRIM(full_name)), CHARINDEX(' ', REVERSE(TRIM(full_name)) + ' ') - 1))), 2, LEN(TRIM(REVERSE(LEFT(REVERSE(TRIM(full_name)), CHARINDEX(' ', REVERSE(TRIM(full_name)) + ' ') - 1)))))) AS full_name,
    
    -- Feature Engineering: Separate merged names into proper cased relational slots
    UPPER(LEFT(TRIM(LEFT(TRIM(full_name), CHARINDEX(' ', TRIM(full_name) + ' ') - 1)), 1)) + 
    LOWER(SUBSTRING(TRIM(LEFT(TRIM(full_name), CHARINDEX(' ', TRIM(full_name) + ' ') - 1)), 2, LEN(TRIM(LEFT(TRIM(full_name), CHARINDEX(' ', TRIM(full_name) + ' ') - 1))))) AS first_name,
    
    UPPER(LEFT(TRIM(REVERSE(LEFT(REVERSE(TRIM(full_name)), CHARINDEX(' ', REVERSE(TRIM(full_name)) + ' ') - 1))), 1)) + 
    LOWER(SUBSTRING(TRIM(REVERSE(LEFT(REVERSE(TRIM(full_name)), CHARINDEX(' ', REVERSE(TRIM(full_name)) + ' ') - 1))), 2, LEN(TRIM(REVERSE(LEFT(REVERSE(TRIM(full_name)), CHARINDEX(' ', REVERSE(TRIM(full_name)) + ' ') - 1)))))) AS last_name,
    
    -- Data Cleaning: Programmatically untangle mixed regional date configurations
    CASE 
        WHEN dob LIKE '%/%/%' AND CHARINDEX('/', dob) = 5 THEN CONVERT(DATE, dob, 111) -- YYYY/MM/DD
        WHEN dob LIKE '%/%/%' AND CHARINDEX('/', dob) <= 3 THEN CONVERT(DATE, dob, 101) -- MM/DD/YYYY
        WHEN dob LIKE '%-%-%' THEN CONVERT(DATE, dob, 105) -- DD-MM-YYYY
        ELSE NULL 
    END AS date_of_birth,
    
    -- Data Cleaning: Normalize loose categorical tokens (M, Male, F, Female)
    CASE 
        WHEN UPPER(LEFT(LTRIM(gender), 1)) = 'M' THEN 'M'
        WHEN UPPER(LEFT(LTRIM(gender), 1)) = 'F' THEN 'F'
        ELSE 'U' 
    END AS gender,
    
    -- Data Cleaning: Map insurance provider blanks to uniform fallbacks
    CASE 
        WHEN UPPER(TRIM(ISNULL(insurance_provider, ''))) IN ('UNKNOWN', 'N/A', '') THEN 'Uninsured' 
        ELSE UPPER(TRIM(insurance_provider)) 
    END AS insurance_provider,
    GETDATE(),
    GETDATE()
FROM dbo.patient_registry
WHERE patient_id IS NOT NULL;
GO

-- Data Cleaning Enhancement: Build or modify the virtual presentation layer with complete Proper Case standardization
CREATE OR ALTER VIEW dbo.vw_dim_patient_registry_presentation AS
WITH RankedPatients AS (
    SELECT 
        patient_key,
        patient_id,
        full_name,
        first_name,
        last_name,
        date_of_birth,
        gender,
        -- Proper Case Insurance Provider: Capitalize the first letter and lowercase the rest
        CASE 
            WHEN UPPER(TRIM(ISNULL(insurance_provider, ''))) IN ('UNKNOWN', 'N/A', '', 'UNINSURED') THEN 'Uninsured' 
            ELSE UPPER(LEFT(TRIM(insurance_provider), 1)) + LOWER(SUBSTRING(TRIM(insurance_provider), 2, LEN(TRIM(insurance_provider))))
        END AS insurance_provider,
        ROW_NUMBER() OVER (
            PARTITION BY full_name, date_of_birth 
            ORDER BY patient_key ASC
        ) AS row_num
    FROM dbo.dim_patient_registry
)
SELECT 
    patient_key,
    patient_id,
    full_name,
    first_name,
    last_name,
    date_of_birth,
    gender,
    insurance_provider
FROM RankedPatients
WHERE row_num = 1;
GO
```

This pass rebuilds full_name, first_name, and last_name in proper case, so MARY SMITH, mary smith, and Mary SMITH all collapse into one consistent Mary Smith rather than three visually different values. It also adds a presentation view on top of the base table, which is where the real duplicate problem gets solved correctly. Two patients can legitimately share the same name, so the view never treats a matching full_name alone as a duplicate. It only collapses a row down when both full_name and date_of_birth match together, keeping the earliest patient_key and dropping the rest, which correctly separates true duplicate entries from patients who are simply different people with a common name.

### Cleaning the Billing Invoice Table

```sql
--Clean billing invoice table
DELETE FROM dbo.fact_billing_invoices;
GO

INSERT INTO dbo.fact_billing_invoices (
    invoice_id, 
    patient_key, 
    patient_id, 
    billing_date, 
    amount_charged, 
    payment_status
)
SELECT 
    UPPER(TRIM(b.invoice_id)),
    p.patient_key, 
    b.patient_id,
    CASE 
        WHEN b.billing_date LIKE '%/%/%' THEN CONVERT(DATE, b.billing_date, 103) 
        ELSE CONVERT(DATE, b.billing_date, 120) 
    END,
    TRY_CAST(LTRIM(RTRIM(REPLACE(REPLACE(REPLACE(b.amount_charged, '$', ''), 'USD', ''), ' ', ''))) AS DECIMAL(10,2)),
    CASE 
        WHEN LTRIM(RTRIM(b.payment_status)) = '' OR b.payment_status IS NULL THEN 'Pending'
        ELSE UPPER(LEFT(LTRIM(b.payment_status), 1)) + LOWER(SUBSTRING(LTRIM(b.payment_status), 2, LEN(b.payment_status)))
    END
FROM dbo.billing_invoices_raw b
INNER JOIN dbo.dim_patient_registry p ON b.patient_id = p.patient_id 
WHERE b.invoice_id IS NOT NULL;
GO
```

Every payment status is forced into consistent title case, so paid, PAID, and Paid collapse into one predictable value instead of fracturing every group by clause downstream. Any blank or missing status defaults safely to pending rather than vanishing into an unhandled null.

### Cleaning the Clinical Telemetry Table

```sql
--clean clinical telemetry table
-- 1. DELETE ANY EMPTY CONTAINERS TO PREVENT DUPLICATES
DELETE FROM dbo.fact_clinical_telemetry;
GO

-- 2. RELOAD THE BASE ROWS AND EXTRACT THE VITAL SIGN SPLITS
INSERT INTO dbo.fact_clinical_telemetry (
    telemetry_id, 
    patient_key, 
    patient_id, 
    admission_timestamp, 
    doctor_seen_timestamp, 
    systolic_bp, 
    diastolic_bp, 
    readmitted
)
SELECT 
    UPPER(TRIM(t.telemetry_id)),
    p.patient_key,
    t.patient_id,
    TRY_CAST(t.admission_timestamp AS DATETIME),
    TRY_CAST(t.doctor_seen_timestamp AS DATETIME),
    CASE WHEN t.blood_pressure LIKE '%/%' THEN TRY_CAST(LEFT(t.blood_pressure, CHARINDEX('/', t.blood_pressure) - 1) AS INT) ELSE NULL END,
    CASE WHEN t.blood_pressure LIKE '%/%' THEN TRY_CAST(SUBSTRING(t.blood_pressure, CHARINDEX('/', t.blood_pressure) + 1, LEN(t.blood_pressure)) AS INT) ELSE NULL END,
    CASE WHEN t.readmitted = 1 THEN 'Y' ELSE 'N' END
FROM dbo.clinical_telemetry_raw t
INNER JOIN dbo.dim_patient_registry p ON t.patient_key = p.patient_key 
WHERE t.telemetry_id IS NOT NULL 
  AND t.blood_pressure LIKE '%/%';
GO
```

This pass only splits a blood pressure reading when a forward slash actually exists in the text, rather than assuming every row is perfectly formatted. It also standardizes the readmission flag from a raw bit value down to one clean letter, Y or N, that every later query can rely on directly.

---

## Feature Engineering

Four engineered values were prototyped on sample inputs before being written permanently into the pipeline, each one turning a plain raw field into a genuine analytical metric.

### Patient Age and Demographic Tier

```sql
--FEATURE ENGINEERING PHASE:

-- FEATURE METRIC 1 & 2: DYNAMIC PATIENT AGE COMPUTE & CATEGORICAL BINNING MATRIX
-- Description: Continuous age derivation and conditional demographic tier grouping.

-- Declare a test input variable (No table attached)
DECLARE @TestDateOfBirth DATE = '1995-04-15';

SELECT 
    @TestDateOfBirth AS input_dob,
    
    -- Feature 1: Pure Continuous Age Calculation Logic
    DATEDIFF(YEAR, @TestDateOfBirth, GETDATE()) AS engineered_age_in_years,
    
    -- Feature 2: Pure Categorical Binning Matrix Logic
    CASE 
        WHEN DATEDIFF(YEAR, @TestDateOfBirth, GETDATE()) < 18 THEN 'Pediatric (Under 18)'
        WHEN DATEDIFF(YEAR, @TestDateOfBirth, GETDATE()) BETWEEN 18 AND 35 THEN 'Young Adult (18-35)'
        WHEN DATEDIFF(YEAR, @TestDateOfBirth, GETDATE()) BETWEEN 36 AND 60 THEN 'Adult (36-60)'
        ELSE 'Senior (Over 60)'
    END AS engineered_demographic_tier;
```

A raw date of birth alone tells an administrator very little. Turning it into an age in years, then folding that age into a named tier such as pediatric, young adult, adult, or senior, gives every later readmission query a clean, groupable category to work with.

### Financial Aging Interval

```sql
-- Declare a test input variable (No table attached)
DECLARE @TestInvoiceDate DATE = DATEADD(DAY, -45, GETDATE());

SELECT 
    @TestInvoiceDate AS input_billing_date,
    
    -- Feature 3: Pure Financial Aging Interval Logic
    DATEDIFF(DAY, @TestInvoiceDate, GETDATE()) AS engineered_days_overdue;
```

An invoice date on its own is just a timestamp. Measuring the distance between that date and today turns it into an aging signal, exactly the number a finance team needs to know how urgently a claim needs chasing.

### Clinical Response Delay

```sql
-- FEATURE METRIC 3: DYNAMIC FINANCIAL AGING & INVOICE OVERDUE TRACKER
-- Description: Time-series interval calculation tracking outstanding ledger durations.

-- Declare test input variables (No table attached)
DECLARE @TestAdmissionTime DATETIME = '2026-09-25 08:00:00';
DECLARE @TestDoctorSeenTime DATETIME = '2026-09-25 11:30:00';

SELECT 
    @TestAdmissionTime AS input_admission_time,
    @TestDoctorSeenTime AS input_doctor_seen_time,
    
    -- Feature 4: Pure Operational SLA Delay Logic
    DATEDIFF(HOUR, @TestAdmissionTime, @TestDoctorSeenTime) AS engineered_time_to_doctor_hours;
```

Two raw timestamps mean little sitting side by side. Calculating the gap between admission and doctor assessment in hours turns them into a genuine operational metric, one that can be measured against a service threshold and tracked over time. All three engineered values, age, days overdue, and hours to doctor, were carried forward directly into the schema below.

---

## Data Modeling and Relational Schema

With clean data and engineered features ready, this phase gives everything a proper home, enforcing relationships so no future insert can break the connection between a patient and their records.

```sql
--Create primary key and foreign key relationship table phase:
--Create billing fact table with explicit key connections
IF OBJECT_ID('dbo.fact_billing_invoices', 'U') IS NOT NULL 
    DROP TABLE dbo.fact_billing_invoices;
GO

CREATE TABLE dbo.fact_billing_invoices (
    billing_key INT IDENTITY(1,1) PRIMARY KEY, -- Primary Key Anchor
    invoice_id VARCHAR(15) NOT NULL UNIQUE,
    patient_key INT NOT NULL,                  -- Relational Key Column
    patient_id VARCHAR(10) NOT NULL,
    billing_date DATE NOT NULL,
    amount_charged DECIMAL(10,2) NOT NULL,
    payment_status VARCHAR(50) NOT NULL,
    days_overdue AS (DATEDIFF(DAY, billing_date, GETDATE())),
    
    -- Formally enforce the Foreign Key relationship line down to the parent dimension lookup
    CONSTRAINT fk_billing_to_patient 
        FOREIGN KEY (patient_key) 
        REFERENCES dbo.dim_patient_registry(patient_key)
);
GO


--Create clinical fact table with explicit key connections
IF OBJECT_ID('dbo.fact_clinical_telemetry', 'U') IS NOT NULL 
    DROP TABLE dbo.fact_clinical_telemetry;
GO

CREATE TABLE dbo.fact_clinical_telemetry (
    telemetry_key INT IDENTITY(1,1) PRIMARY KEY, -- Primary Key Anchor
    telemetry_id VARCHAR(15) NOT NULL UNIQUE,
    patient_key INT NOT NULL,                    -- Relational Key Column
    patient_id VARCHAR(10) NOT NULL,
    admission_timestamp DATETIME NOT NULL,
    doctor_seen_timestamp DATETIME NULL,
    time_to_doctor_hours AS (DATEDIFF(HOUR, admission_timestamp, doctor_seen_timestamp)),
    systolic_bp INT NULL,
    diastolic_bp INT NULL,
    readmitted CHAR(1) NOT NULL,
    
    -- Formally enforce the Foreign Key relationship line down to the parent dimension lookup
    CONSTRAINT fk_clinical_to_patient 
        FOREIGN KEY (patient_key) 
        REFERENCES dbo.dim_patient_registry(patient_key)
);
GO
```

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/d9abb496-60a4-4b46-a6d3-24f2fe32405a" />


The schema follows a simple star pattern. One dimension table, dim_patient_registry, holds identity and demographic detail at the center. Two fact tables radiate outward, each tied back through a formal foreign key on patient_key. Days overdue and hours to doctor are computed columns, recalculated live on every query rather than stored values, and the foreign key constraint makes an orphaned record structurally impossible from this point forward.

---

## Identify Revenue Leakage Pathways

```sql
--BUSINESS OBJECTIVES ANALYSIS PHASE:
--1. INSURANCE REVENUE LEAKAGE MATRIX
-- Objective: Quantify cash flow gridlocks across payment status states.
WITH AggregatedInsuranceLedger AS (
    SELECT 
        p.insurance_provider,
        -- Calculate total invoice volume issued per provider group
        COUNT(b.billing_key) AS total_invoices_issued,
        
        -- Aggregate the absolute revenue safely injected into corporate cash flow
        SUM(CASE WHEN b.payment_status = 'Paid' THEN b.amount_charged ELSE 0 END) AS total_collected_revenue,
        
        -- Quantify active leakage: Cash stalled in an overdue collector state
        SUM(CASE WHEN b.payment_status = 'Overdue' THEN b.amount_charged ELSE 0 END) AS total_overdue_leakage,
        
        -- Quantify disputed leakage: Cash locked in administrative audits or legal blocks
        SUM(CASE WHEN b.payment_status = 'Disputed' THEN b.amount_charged ELSE 0 END) AS total_disputed_leakage,
        
        -- Aggregate the gross financial footprint across all transactional accounts
        SUM(b.amount_charged) AS gross_total_billed_amount
    FROM dbo.fact_billing_invoices b
    INNER JOIN dbo.dim_patient_registry p ON b.patient_key = p.patient_key
    GROUP BY p.insurance_provider
)
SELECT 
    insurance_provider,
    total_invoices_issued,
    
    -- Format numeric scales cleanly into math-ready decimal currency metrics
    CAST(gross_total_billed_amount AS DECIMAL(10,2)) AS gross_billed_revenue,
    CAST(total_collected_revenue AS DECIMAL(10,2)) AS actual_collected_cash,
    
    -- Expose the clear operational leakage components
    CAST(total_overdue_leakage AS DECIMAL(10,2)) AS stalled_overdue_revenue,
    CAST(total_disputed_leakage AS DECIMAL(10,2)) AS contested_disputed_revenue,
    
    -- Feature Engineering: Calculate the corporate risk rate percentage per provider
    CAST(
        ((total_overdue_leakage + total_disputed_leakage) * 100.0) / NULLIF(gross_total_billed_amount, 0) 
        AS DECIMAL(5,2)
    ) AS total_revenue_leakage_rate_pct
FROM AggregatedInsuranceLedger
ORDER BY stalled_overdue_revenue DESC; 
GO
```

This query splits every dollar billed into three honest buckets per insurer, collected, overdue, and disputed. Adding overdue and disputed together, then dividing by the gross amount billed, produces one revenue leakage rate per provider. Sorting by overdue balance instantly surfaces which insurance relationships are causing the most financial pain, replacing a wall of raw invoice rows with a single readable ranking.

---

## Audit Clinical Wait Time SLA Breaches

```sql
--2. EMERGENCY TRIAGE SLA COMPLIANCE AUDIT
-- Objective: Quantify patient wait-time breaches exceeding the 3-hour threshold.
WITH ClinicalResponseBase AS (
    SELECT 
        t.telemetry_key,
        p.gender AS normalized_gender,
        p.insurance_provider,
        t.admission_timestamp,
        t.doctor_seen_timestamp,
        
        -- Pull the pre-calculated hourly lag between check-in and doctor assessment
        t.time_to_doctor_hours,
        
        -- Feature Engineering: Flag individual rows that break the 3-hour SLA barrier
        CASE 
            WHEN t.time_to_doctor_hours > 3 THEN 1 
            ELSE 0 
        END AS is_sla_breach
    FROM dbo.fact_clinical_telemetry t
    INNER JOIN dbo.dim_patient_registry p ON t.patient_key = p.patient_key
    -- Focus exclusively on patients who have completed triage and seen a doctor
    WHERE t.doctor_seen_timestamp IS NOT NULL
)
SELECT 
    normalized_gender,
    COUNT(telemetry_key) AS total_patient_encounters,
    
    -- Aggregate total breach volumes using defensive conditional counting
    SUM(is_sla_breach) AS total_sla_breaches,
    
    -- Calculate precise absolute workflow average wait times per cohort
    CAST(AVG(CAST(time_to_doctor_hours AS DECIMAL(10,2))) AS DECIMAL(5,2)) AS average_wait_hours,
    
    -- Isolate worst-case delay outliers for executive risk mitigation
    MAX(time_to_doctor_hours) AS peak_delay_hours_logged,
    
    -- Feature Engineering: Compute the operational breach rate percentage
    CAST(
        (SUM(is_sla_breach) * 100.0) / NULLIF(COUNT(telemetry_key), 0) 
        AS DECIMAL(5,2)
    ) AS sla_breach_rate_pct
FROM ClinicalResponseBase
GROUP BY normalized_gender
ORDER BY sla_breach_rate_pct DESC;
GO
```

A three hour threshold marks the point past which a wait becomes a genuine care risk. Every completed encounter is flagged yes or no against that line, then rolled up by gender to reveal average wait, worst recorded delay, and an overall breach rate per group. Sorting by breach rate turns a vague operational worry into one specific, actionable number.

---

## Profile Demographic Care Vulnerabilities

```sql
--3. DEMOGRAPHIC PATIENT READMISSION ANALYSIS
-- Objective: Profile clinical return risk rates across targeted age brackets.
WITH PatientDemographicCohort AS (
    SELECT 
        t.telemetry_key,
        p.patient_id,
        p.age_in_years,
        -- Operational Parameter: Standardize the binary character indicator
        t.readmitted,
        
        -- Feature Engineering: Categorical Binning Matrix to build corporate age cohorts
        CASE 
            WHEN p.age_in_years < 18 THEN 'Pediatric (Under 18)'
            WHEN p.age_in_years BETWEEN 18 AND 35 THEN 'Young Adult (18-35)'
            WHEN p.age_in_years BETWEEN 36 AND 60 THEN 'Adult (36-60)'
            ELSE 'Senior (Over 60)'
        END AS demographic_tier
    FROM dbo.fact_clinical_telemetry t
    INNER JOIN dbo.dim_patient_registry p ON t.patient_key = p.patient_key
)
SELECT 
    demographic_tier,
    -- Quantify total patient admission volume per age segment
    COUNT(telemetry_key) AS total_admissions,
    
    -- Isolate absolute return volumes utilizing conditional summation overrides
    SUM(CASE WHEN readmitted = 'Y' THEN 1 ELSE 0 END) AS absolute_readmission_volume,
    
    -- Feature Engineering: Compute the analytical readmission rate percentage
    CAST(
        (SUM(CASE WHEN readmitted = 'Y' THEN 1 ELSE 0 END) * 100.0) / NULLIF(COUNT(telemetry_key), 0) 
        AS DECIMAL(5,2)
    ) AS readmission_rate_pct
FROM PatientDemographicCohort
GROUP BY demographic_tier
ORDER BY readmission_rate_pct DESC;
GO

USE Health_Analytics;
GO
```

Every encounter is sorted into pediatric, young adult, adult, or senior using the demographic tier engineered earlier. Each tier is measured on total admissions against total readmissions, producing a clean readmission rate per age group. That ranking shows leadership exactly which demographic carries the highest clinical return risk, and where a follow up program would do the most good.

---

## Track Triage Operational Efficiency

```sql
--4. TRIAGE TIMELINE OPERATIONAL EFFICIENCY
-- Objective: Calculate a rolling 5-encounter moving average to smooth wait trends.
WITH OrderedTriageTimeline AS (
    SELECT 
        t.telemetry_key,
        t.admission_timestamp,
        p.patient_id,
        p.full_name,
        p.gender AS normalized_gender,
        t.time_to_doctor_hours,
        
        -- Advanced Feature Engineering: Compute chronological rolling 5-encounter wait average
        -- This window function smooths out random spikes by tracking a patient and their 4 predecessors
        AVG(CAST(t.time_to_doctor_hours AS DECIMAL(10,2))) OVER (
            PARTITION BY p.gender 
            ORDER BY t.admission_timestamp ASC
            ROWS BETWEEN 4 PRECEDING AND CURRENT ROW
        ) AS rolling_avg_wait_hours
    FROM dbo.fact_clinical_telemetry t
    INNER JOIN dbo.dim_patient_registry p ON t.patient_key = p.patient_key
    WHERE t.doctor_seen_timestamp IS NOT NULL
)
SELECT TOP 20
    telemetry_key,
    admission_timestamp,
    patient_id,
    full_name,
    normalized_gender,
    time_to_doctor_hours AS actual_wait_hours,
    
    -- Format the moving feature cleanly into an executive-ready numeric metric
    CAST(rolling_avg_wait_hours AS DECIMAL(5,2)) AS smoothed_rolling_avg_hours,
    
    -- Feature Engineering: Quantify workflow variance (Delta gap between actual vs trend)
    CAST(
        CAST(time_to_doctor_hours AS DECIMAL(10,2)) - rolling_avg_wait_hours 
        AS DECIMAL(5,2)
    ) AS efficiency_variance_delta
FROM OrderedTriageTimeline
ORDER BY admission_timestamp DESC; 
GO
```

Instead of one flat average, this query builds a rolling average recalculated for every encounter using its four immediate predecessors within the same gender group, ordered by admission time. That smoothed trend reveals the pattern hiding beneath the noise. Comparing each actual wait against the trend, the efficiency variance, flags exactly where triage suddenly improved or slipped, a sharper early warning signal than any single average.

---

## Evaluate Insurance Collection Aging Metrics

```sql
--5. INSURANCE COLLECTION AGING MATRIX & RANKING
-- Objective: Calculate average outstanding aging loops and rank collection speeds.
WITH InsuranceAgingSummary AS (
    SELECT 
        p.insurance_provider,
        COUNT(b.billing_key) AS total_claims_tracked,
        
        -- Aggregate outstanding invoice backlogs stuck in an Overdue state
        SUM(CASE WHEN b.payment_status = 'Overdue' THEN 1 ELSE 0 END) AS total_overdue_invoices,
        SUM(CASE WHEN b.payment_status = 'Overdue' THEN b.amount_charged ELSE 0 END) AS total_overdue_cash_leakage,
        
        -- Calculate absolute average aging duration across all outstanding receivables
        AVG(CAST(b.days_overdue AS DECIMAL(10,2))) AS average_days_outstanding
    FROM dbo.fact_billing_invoices b
    INNER JOIN dbo.dim_patient_registry p ON b.patient_key = p.patient_key
    GROUP BY p.insurance_provider
)
SELECT 
    insurance_provider,
    total_claims_tracked,
    total_overdue_invoices,
    CAST(total_overdue_cash_leakage AS DECIMAL(10,2)) AS gross_overdue_balance,
    CAST(average_days_outstanding AS DECIMAL(5,1)) AS avg_days_claims_outstanding,
    
    -- Feature Engineering: Deploy dense ranking to build an executive insurance collection leaderboard
    DENSE_RANK() OVER (ORDER BY average_days_outstanding ASC) AS insurer_collection_speed_rank
FROM InsuranceAgingSummary
ORDER BY insurer_collection_speed_rank ASC; 
GO
```

Every insurer is measured against the days overdue feature built directly into the billing schema, giving a genuine average claim age rather than just a raw dollar total. A dense rank sits on top of that average, building a leaderboard from fastest to slowest paying insurer with no gaps, even where two insurers tie exactly. That format turns a wall of numbers into a ten second read for any finance director.

---

## Key Findings and Insights

Revenue leakage is never evenly spread across insurers, the leakage rate calculation in objective one exists specifically to expose which providers carry a disproportionate share of overdue and disputed cash.

Triage delays are not a single flat problem, grouping by gender and layering a rolling average on top catches localized slowdowns that one hospital wide average would otherwise hide.

Readmission risk clusters clearly by age tier, any bracket running meaningfully hotter than the others becomes immediately visible once bucketed, rather than buried inside one blended number.

Collection speed and collection risk are related but distinct, an insurer can carry a large overdue balance while still paying reasonably fast on average, or the reverse, and separating those two metrics keeps both problems honest and visible side by side.

---

## Recommendations

Insurers surfacing at the top of the leakage and aging leaderboards deserve a direct conversation, backed by the exact overdue dollar figures and average settlement time this pipeline produces on demand.

Any cohort showing a service level breach rate meaningfully above the rest deserves a focused staffing or triage workflow review, using the peak delay and average wait figures as starting evidence.

The age tier carrying the highest readmission rate is the natural first candidate for a discharge follow up program, since that investment statistically touches the largest number of preventable returns.

This pipeline was built to be re-run, not read once and filed away, scheduling it on a regular cadence turns all five objectives into a living operational dashboard rather than a one time report.

---

## Conclusion

Three messy, unreliable tables became a fully modeled, referentially sound analytics pipeline capable of answering five real financial and clinical questions on demand. Every anomaly from the audit phase was traced through to a specific fix in transformation and cleaning, every engineered feature found a permanent home in the final schema, and every business objective now runs as a clean, repeatable query. That is the real work of a data analyst, building the entire chain of trust underneath an answer so it actually means something to the people who have to act on it.
