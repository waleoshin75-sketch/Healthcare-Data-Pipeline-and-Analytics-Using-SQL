# Healthcare Data Pipeline and Analytics Using SQL Server

**T-SQL | Star schema | Window functions | Revenue leakage, SLA compliance and readmission analytics**

Built an end-to-end SQL Server pipeline that audits, cleans and models raw hospital records, then answers five business questions on revenue leakage, wait-time compliance, readmissions and collections.

| | |
|---|---|
| **Tools** | SQL Server, T-SQL, SSMS |
| **Data** | 3 raw tables rebuilt as 1 dimension and 2 fact tables |
| **Deliverable** | Reusable cleaning scripts, PK/FK schema and 5 analytical queries |

---

## Table of Contents
1. [Project Overview](#1-project-overview)
2. [Dataset Overview](#2-dataset-overview)
3. [Data Quality Assessment](#3-data-quality-assessment)
4. [Data Cleaning and Error Correction](#4-data-cleaning-and-error-correction)
5. [Feature Engineering](#5-feature-engineering)
6. [Data Modeling](#6-data-modeling)
7. [Business Objectives](#7-business-objectives)
8. [Data Visualization and Interpretation](#8-data-visualization-and-interpretation)
9. [Key Findings and Insights](#8-key-findings-and-insight)
10. [Recommendations](#9-recommendations)
11. [Conclusion](#10-conclusion)

---

## 1. Project Overview
Hospital data arrives with duplicate patients, mixed date formats, text-contaminated billing amounts and orphaned records. This project fixes those issues in T-SQL, enforces referential integrity and uses the clean model to answer five objectives:

1. Insurance revenue leakage
2. Emergency triage SLA compliance
3. Readmission risk by demographic tier
4. Triage wait-time efficiency trends
5. Insurance collection aging and ranking

## 2. Dataset Overview

| Table | Type | Contents |
|---|---|---|
| `dim_patient_registry` | Dimension | Patient ID, name, date of birth, gender, insurance provider |
| `fact_billing_invoices` | Fact | Invoice, billing date, amount charged, payment status |
| `fact_clinical_telemetry` | Fact | Admission and doctor-seen timestamps, blood pressure, readmission flag |

Raw staging tables (`patient_registry_raw`, `billing_invoices_raw`, `clinical_telemetry_raw`) are preserved untouched.

## 3. Data Quality Assessment

| Check | Method | Issue found |
|---|---|---|
| Duplicates | `COUNT(*)` vs `COUNT(DISTINCT key)`, grouping by `full_name` | Repeated patient profiles in the registry |
| Missing and blank values | `NULL` and `LTRIM(RTRIM())` tests per column | Blank names, genders and insurance providers |
| Hidden spaces | `LEN(full_name)` vs `LEN(TRIM(full_name))` | Leading and trailing spaces in names |
| Placeholders | `IN ('UNKNOWN','N/A','')` on `insurance_provider` | Inconsistent unknown labels |
| Date contamination | `TRY_CAST(dob AS DATE) IS NULL` | Mixed regional date formats |
| Currency text | `TRY_CAST` after stripping `$`, `USD` | Amounts stored as text with symbols |
| Negative values | `LIKE '-%'` and `< 0` test on amounts | Impossible negative charges |
| Orphan keys | `NOT IN` against the registry | Billing and telemetry rows with no matching patient |

```sql
-- Example audit: row count vs unique business keys
SELECT 'patient_registry' AS table_name, COUNT(*) AS total_rows, COUNT(DISTINCT patient_id) AS unique_business_keys
FROM dbo.patient_registry
UNION ALL
SELECT 'billing_invoices', COUNT(*), COUNT(DISTINCT invoice_id) FROM dbo.billing_invoices
UNION ALL
SELECT 'clinical_telemetry', COUNT(*), COUNT(DISTINCT telemetry_id) FROM dbo.clinical_telemetry;
```

## 4. Data Cleaning and Error Correction

Cleaning loads the raw tables into the typed dimension and fact tables. Facts join to the dimension, so orphan records are excluded.

### Patient registry
Trims text, splits names, untangles three date formats, normalizes gender and replaces blank insurance values with `Uninsured`.

```sql
INSERT INTO dbo.dim_patient_registry (
    patient_id, full_name, first_name, last_name, date_of_birth,
    gender, insurance_provider, record_created_date, record_updated_date
)
SELECT
    UPPER(TRIM(patient_id)),
    TRIM(full_name),
    TRIM(LEFT(TRIM(full_name), CHARINDEX(' ', TRIM(full_name) + ' ') - 1)),
    TRIM(REVERSE(LEFT(REVERSE(TRIM(full_name)), CHARINDEX(' ', REVERSE(TRIM(full_name)) + ' ') - 1))),
    CASE
        WHEN dob LIKE '%/%/%' AND CHARINDEX('/', dob) = 5  THEN CONVERT(DATE, dob, 111) -- YYYY/MM/DD
        WHEN dob LIKE '%/%/%' AND CHARINDEX('/', dob) <= 3 THEN CONVERT(DATE, dob, 101) -- MM/DD/YYYY
        WHEN dob LIKE '%-%-%' THEN CONVERT(DATE, dob, 105)                              -- DD-MM-YYYY
        ELSE NULL
    END,
    CASE
        WHEN UPPER(LEFT(LTRIM(gender), 1)) = 'M' THEN 'M'
        WHEN UPPER(LEFT(LTRIM(gender), 1)) = 'F' THEN 'F'
        ELSE 'U'
    END,
    CASE
        WHEN UPPER(TRIM(ISNULL(insurance_provider, ''))) IN ('UNKNOWN', 'N/A', '') THEN 'Uninsured'
        ELSE UPPER(TRIM(insurance_provider))
    END,
    GETDATE(),
    GETDATE()
FROM dbo.patient_registry
WHERE patient_id IS NOT NULL;
```

### Billing invoices
Strips currency symbols, converts amounts to `DECIMAL(10,2)`, standardizes payment status casing and defaults blanks to `Pending`.

```sql
INSERT INTO dbo.fact_billing_invoices (invoice_id, patient_key, patient_id, billing_date, amount_charged, payment_status)
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
```

### Clinical telemetry
Splits blood pressure into systolic and diastolic values and converts the readmission flag to `Y` or `N`.

```sql
INSERT INTO dbo.fact_clinical_telemetry (
    telemetry_id, patient_key, patient_id, admission_timestamp,
    doctor_seen_timestamp, systolic_bp, diastolic_bp, readmitted
)
SELECT
    UPPER(TRIM(t.telemetry_id)),
    p.patient_key,
    t.patient_id,
    TRY_CAST(t.admission_timestamp AS DATETIME),
    TRY_CAST(t.doctor_seen_timestamp AS DATETIME),
    TRY_CAST(LEFT(t.blood_pressure, CHARINDEX('/', t.blood_pressure) - 1) AS INT),
    TRY_CAST(SUBSTRING(t.blood_pressure, CHARINDEX('/', t.blood_pressure) + 1, LEN(t.blood_pressure)) AS INT),
    CASE WHEN t.readmitted = 1 THEN 'Y' ELSE 'N' END
FROM dbo.clinical_telemetry_raw t
INNER JOIN dbo.dim_patient_registry p ON t.patient_id = p.patient_id
WHERE t.telemetry_id IS NOT NULL
  AND t.blood_pressure LIKE '%/%';
```

### Duplicate patients (presentation view)
A view keeps one record per patient name and date of birth, so duplicates never reach reports.

```sql
CREATE VIEW dbo.vw_dim_patient_registry_presentation AS
SELECT *
FROM (
    SELECT *,
           ROW_NUMBER() OVER (PARTITION BY full_name, date_of_birth ORDER BY patient_key) AS rn
    FROM dbo.dim_patient_registry
) d
WHERE rn = 1;
```

> Name casing is standardized with the proper-case function applied to `full_name`, `first_name` and `last_name` in the registry load.

## 5. Feature Engineering

Calculated features are written as reusable logic, tested here with sample inputs.

```sql
-- Age and demographic tier
DECLARE @TestDateOfBirth DATE = '1995-04-15';
SELECT
    DATEDIFF(YEAR, @TestDateOfBirth, GETDATE()) AS engineered_age_in_years,
    CASE
        WHEN DATEDIFF(YEAR, @TestDateOfBirth, GETDATE()) < 18 THEN 'Pediatric (Under 18)'
        WHEN DATEDIFF(YEAR, @TestDateOfBirth, GETDATE()) BETWEEN 18 AND 35 THEN 'Young Adult (18-35)'
        WHEN DATEDIFF(YEAR, @TestDateOfBirth, GETDATE()) BETWEEN 36 AND 60 THEN 'Adult (36-60)'
        ELSE 'Senior (Over 60)'
    END AS engineered_demographic_tier;

-- Days outstanding on an invoice
DECLARE @TestInvoiceDate DATE = DATEADD(DAY, -45, GETDATE());
SELECT DATEDIFF(DAY, @TestInvoiceDate, GETDATE()) AS engineered_days_overdue;

-- Wait time from admission to doctor
DECLARE @TestAdmissionTime DATETIME = '2026-09-25 08:00:00';
DECLARE @TestDoctorSeenTime DATETIME = '2026-09-25 11:30:00';
SELECT DATEDIFF(HOUR, @TestAdmissionTime, @TestDoctorSeenTime) AS engineered_time_to_doctor_hours;
```

| Feature | Logic | Stored as |
|---|---|---|
| Patient age | `DATEDIFF(YEAR, date_of_birth, GETDATE())` | Calculated in analysis queries |
| Demographic tier | Pediatric, Young Adult, Adult, Senior bins | Calculated in analysis queries |
| `days_overdue` | `DATEDIFF(DAY, billing_date, GETDATE())` | Computed column on `fact_billing_invoices` |
| `time_to_doctor_hours` | `DATEDIFF(HOUR, admission, doctor_seen)` | Computed column on `fact_clinical_telemetry` |

## 6. Data Modeling

Star schema with `dim_patient_registry` as the dimension and two fact tables linked on `patient_key`. Foreign keys enforce that no billing or clinical row exists without a valid patient.

| Fact table | Foreign key | Dimension primary key |
|---|---|---|
| `fact_billing_invoices` | `patient_key` | `dim_patient_registry.patient_key` |
| `fact_clinical_telemetry` | `patient_key` | `dim_patient_registry.patient_key` |

<details>
<summary>View table definitions</summary>

```sql
CREATE TABLE dbo.fact_billing_invoices (
    billing_key INT IDENTITY(1,1) PRIMARY KEY,
    invoice_id VARCHAR(15) NOT NULL UNIQUE,
    patient_key INT NOT NULL,
    patient_id VARCHAR(10) NOT NULL,
    billing_date DATE NOT NULL,
    amount_charged DECIMAL(10,2) NOT NULL,
    payment_status VARCHAR(50) NOT NULL,
    days_overdue AS (DATEDIFF(DAY, billing_date, GETDATE())),
    CONSTRAINT fk_billing_to_patient
        FOREIGN KEY (patient_key) REFERENCES dbo.dim_patient_registry(patient_key)
);

CREATE TABLE dbo.fact_clinical_telemetry (
    telemetry_key INT IDENTITY(1,1) PRIMARY KEY,
    telemetry_id VARCHAR(15) NOT NULL UNIQUE,
    patient_key INT NOT NULL,
    patient_id VARCHAR(10) NOT NULL,
    admission_timestamp DATETIME NOT NULL,
    doctor_seen_timestamp DATETIME NULL,
    time_to_doctor_hours AS (DATEDIFF(HOUR, admission_timestamp, doctor_seen_timestamp)),
    systolic_bp INT NULL,
    diastolic_bp INT NULL,
    readmitted CHAR(1) NOT NULL,
    CONSTRAINT fk_clinical_to_patient
        FOREIGN KEY (patient_key) REFERENCES dbo.dim_patient_registry(patient_key)
);
```
</details>

## 7. Business Analysis

| # | Objective | Technique |
|---|---|---|
| 1 | Insurance revenue leakage | CTE with conditional aggregation by provider |
| 2 | Triage SLA compliance (3-hour threshold) | CTE, breach flag, breach rate by gender |
| 3 | Readmission by demographic tier | CTE with age binning |
| 4 | Triage wait-time trend | Rolling 5-encounter average window function |
| 5 | Collection speed ranking | `DENSE_RANK` over average days outstanding |

<details>
<summary><b>Objective 1: Insurance revenue leakage matrix</b></summary>

Quantifies cash tied up in overdue and disputed invoices per provider.

```sql
WITH AggregatedInsuranceLedger AS (
    SELECT
        p.insurance_provider,
        COUNT(b.billing_key) AS total_invoices_issued,
        SUM(CASE WHEN b.payment_status = 'Paid'     THEN b.amount_charged ELSE 0 END) AS total_collected_revenue,
        SUM(CASE WHEN b.payment_status = 'Overdue'  THEN b.amount_charged ELSE 0 END) AS total_overdue_leakage,
        SUM(CASE WHEN b.payment_status = 'Disputed' THEN b.amount_charged ELSE 0 END) AS total_disputed_leakage,
        SUM(b.amount_charged) AS gross_total_billed_amount
    FROM dbo.fact_billing_invoices b
    INNER JOIN dbo.dim_patient_registry p ON b.patient_key = p.patient_key
    GROUP BY p.insurance_provider
)
SELECT
    insurance_provider,
    total_invoices_issued,
    CAST(gross_total_billed_amount AS DECIMAL(10,2)) AS gross_billed_revenue,
    CAST(total_collected_revenue   AS DECIMAL(10,2)) AS actual_collected_cash,
    CAST(total_overdue_leakage     AS DECIMAL(10,2)) AS stalled_overdue_revenue,
    CAST(total_disputed_leakage    AS DECIMAL(10,2)) AS contested_disputed_revenue,
    CAST(((total_overdue_leakage + total_disputed_leakage) * 100.0)
         / NULLIF(gross_total_billed_amount, 0) AS DECIMAL(5,2)) AS total_revenue_leakage_rate_pct
FROM AggregatedInsuranceLedger
ORDER BY stalled_overdue_revenue DESC;
```
</details>

<details>
<summary><b>Objective 2: Emergency triage SLA compliance audit</b></summary>

Counts patients who waited more than three hours to see a doctor.

```sql
WITH ClinicalResponseBase AS (
    SELECT
        t.telemetry_key,
        p.gender AS normalized_gender,
        t.time_to_doctor_hours,
        CASE WHEN t.time_to_doctor_hours > 3 THEN 1 ELSE 0 END AS is_sla_breach
    FROM dbo.fact_clinical_telemetry t
    INNER JOIN dbo.dim_patient_registry p ON t.patient_key = p.patient_key
    WHERE t.doctor_seen_timestamp IS NOT NULL
)
SELECT
    normalized_gender,
    COUNT(telemetry_key) AS total_patient_encounters,
    SUM(is_sla_breach) AS total_sla_breaches,
    CAST(AVG(CAST(time_to_doctor_hours AS DECIMAL(10,2))) AS DECIMAL(5,2)) AS average_wait_hours,
    MAX(time_to_doctor_hours) AS peak_delay_hours_logged,
    CAST((SUM(is_sla_breach) * 100.0) / NULLIF(COUNT(telemetry_key), 0) AS DECIMAL(5,2)) AS sla_breach_rate_pct
FROM ClinicalResponseBase
GROUP BY normalized_gender
ORDER BY sla_breach_rate_pct DESC;
```
</details>

<details>
<summary><b>Objective 3: Readmission by demographic tier</b></summary>

Age is derived from `date_of_birth` inside the query, then binned into tiers.

```sql
WITH PatientDemographicCohort AS (
    SELECT
        t.telemetry_key,
        t.readmitted,
        CASE
            WHEN DATEDIFF(YEAR, p.date_of_birth, GETDATE()) < 18 THEN 'Pediatric (Under 18)'
            WHEN DATEDIFF(YEAR, p.date_of_birth, GETDATE()) BETWEEN 18 AND 35 THEN 'Young Adult (18-35)'
            WHEN DATEDIFF(YEAR, p.date_of_birth, GETDATE()) BETWEEN 36 AND 60 THEN 'Adult (36-60)'
            ELSE 'Senior (Over 60)'
        END AS demographic_tier
    FROM dbo.fact_clinical_telemetry t
    INNER JOIN dbo.dim_patient_registry p ON t.patient_key = p.patient_key
)
SELECT
    demographic_tier,
    COUNT(telemetry_key) AS total_admissions,
    SUM(CASE WHEN readmitted = 'Y' THEN 1 ELSE 0 END) AS absolute_readmission_volume,
    CAST((SUM(CASE WHEN readmitted = 'Y' THEN 1 ELSE 0 END) * 100.0)
         / NULLIF(COUNT(telemetry_key), 0) AS DECIMAL(5,2)) AS readmission_rate_pct
FROM PatientDemographicCohort
GROUP BY demographic_tier
ORDER BY readmission_rate_pct DESC;
```
</details>

<details>
<summary><b>Objective 4: Triage timeline efficiency (rolling average)</b></summary>

A rolling average over the current and four previous encounters smooths out random spikes.

```sql
WITH OrderedTriageTimeline AS (
    SELECT
        t.telemetry_key,
        t.admission_timestamp,
        p.patient_id,
        p.full_name,
        p.gender AS normalized_gender,
        t.time_to_doctor_hours,
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
    CAST(rolling_avg_wait_hours AS DECIMAL(5,2)) AS smoothed_rolling_avg_hours,
    CAST(CAST(time_to_doctor_hours AS DECIMAL(10,2)) - rolling_avg_wait_hours AS DECIMAL(5,2)) AS efficiency_variance_delta
FROM OrderedTriageTimeline
ORDER BY admission_timestamp DESC;
```
</details>

<details>
<summary><b>Objective 5: Insurance collection aging and ranking</b></summary>

Ranks insurers from fastest to slowest by average days outstanding.

```sql
WITH InsuranceAgingSummary AS (
    SELECT
        p.insurance_provider,
        COUNT(b.billing_key) AS total_claims_tracked,
        SUM(CASE WHEN b.payment_status = 'Overdue' THEN 1 ELSE 0 END) AS total_overdue_invoices,
        SUM(CASE WHEN b.payment_status = 'Overdue' THEN b.amount_charged ELSE 0 END) AS total_overdue_cash_leakage,
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
    DENSE_RANK() OVER (ORDER BY average_days_outstanding ASC) AS insurer_collection_speed_rank
FROM InsuranceAgingSummary
ORDER BY insurer_collection_speed_rank ASC;
```
</details>

### Data Visualization and Interpretation

The cleaned SQL views feed a Power BI dashboard in two pages.

**Executive KPIs**

<img width="2400" height="1357" alt="healthcare-executive-kpis" src="https://github.com/user-attachments/assets/af2bff66-445c-44ed-bdac-29154d07cc10" />

Total billing stands at 1.37M. Pending invoices are the largest status at about 699K, and Uninsured patients account for the most billing at about 416K.

**Business Performance Analysis**

<img width="2400" height="1350" alt="healthcare-business-performance" src="https://github.com/user-attachments/assets/7a161307-7158-4671-8fed-c63330d31093" />

## 8. Key Findings and Insights

Each query returns provider-level, cohort-level and encounter-level results that answer one objective. Results depend on the loaded data, so run the scripts against your dataset to produce figures.

| Objective | Insight the query delivers |
|---|---|
| Revenue leakage | Which insurers hold the largest overdue and disputed balances, and each provider's leakage rate |
| SLA compliance | Breach count, breach rate, average and peak wait by gender cohort |
| Readmission | Which age tier returns most often |
| Triage trend | Whether individual waits run above or below the recent trend |
| Collections | A ranked leaderboard of insurers by average days outstanding |

## 9. Recommendations
1. **Escalate collections with high-leakage insurers:** prioritize providers with the largest overdue and disputed balances.
2. **Staff to the 3-hour SLA:** use breach rates and peak delays to target triage staffing.
3. **Build follow-up programs for high-readmission tiers:** focus discharge planning on the age group with the highest rate.
4. **Monitor the rolling wait average:** alert when actual waits exceed the trend.
5. **Review contracts with slow-paying insurers:** use the collection speed rank in negotiations.

## 10. Conclusion
Auditing the raw tables, enforcing a typed star schema with primary and foreign keys and engineering features at the table level produced a clean base for financial and clinical analytics that can be rerun as new data arrives.

