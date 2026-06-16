# Data Audit & Profiling Report

## Overview

Before any cleaning or transformation, I conducted a full data profiling audit across all six tables using Power Query. The analysis examined null rates, distinct value counts, numeric distributions, date format consistency, and referential integrity between tables.

---

## Table Summaries

| Table | Rows | Columns | Key Issues Found |
|-------|------|---------|-----------------|
| `fact_claims` | 776 | 16 | Nulls in revenue/cost, negative revenue, future dates, duplicate claim IDs |
| `dim_patients` | 618 | 11 | 18 duplicate rows, age outlier (999), region mismatches, structured email nulls |
| `dim_services` | 206 | 8 | 6 duplicate rows, 8-format boolean column |
| `dim_locations` | 80 | 7 | 5 region/state mismatches |
| `dim_providers` | 80 | 7 | 8-format boolean column |
| `bridge_patient_provider` | 785 | 4 | Mixed boolean formats, date format inconsistency |

---

## fact_claims

### Null Rates

| Column | Nulls | Null Rate | Notes |
|--------|-------|-----------|-------|
| `claim_date` | 37 | 4.8% | Cannot be used in time series without imputation or exclusion |
| `revenue` | 52 | 6.7% | Nulls will silently drop from any SUM or AVG |
| `cost` | 51 | 6.6% | Required for profit calculation,  nulls cascade to profit column |
| `copay` | 88 | 11.3% | Self-pay or zero-copay plans |
| `satisfaction_score` | 119 | 15.3% | Highest null rate in table — average score will be skewed if not handled |

### Numeric Distributions

| Column | Min | Max | Mean | Median | Std Dev | p99 | Negatives | Zeros |
|--------|-----|-----|------|--------|---------|-----|-----------|-------|
| `quantity` | 0 | 10 | 5.29 | 5.0 | 2.86 | 10 | 0 | 12 ⚠️ |
| `unit_price` | $52 | $4,982 | $2,546 | $2,611 | $1,391 | $4,943 | 0 | 0 |
| `revenue` | -$35,319 ⚠️ | $809,797 | $18,881 | $10,645 | $48,490 | $257,849 | 13 ⚠️ | 2 ⚠️ |
| `cost` | $70 | $33,151 | $6,594 | $4,932 | $5,749 | $25,868 | 0 | 0 |
| `copay` | $0.71 | $499.99 | $252.58 | $258.27 | $145.50 | $498.15 | 0 | 0 |
| `satisfaction_score` | 1 | 10 | 5.42 | 5.0 | 2.89 | 10 | 0 | 0 |

**Revenue finding:** The mean ($18,881) is nearly 2× the median ($10,645) and the standard deviation ($48,490) is 2.6× the mean strong right skew caused by outlier spikes above the 99th percentile. 13 negative values and 2 zero values on Completed claims are business rule violations requiring correction before any financial aggregation.

**Quantity zeros:** 12 rows have `quantity = 0`. Cross-referencing with `claim_status = 'Completed'` reveals business rule violations. A completed claim cannot have zero units.

### Categorical Distributions

| Column | Distinct Values | Breakdown |
|--------|----------------|-----------|
| `claim_status` | 4 | Completed 54% · Pending 20% · Denied 15% · Under Review 11% |
| `insurance_type` | 5 | Self-Pay 21% · Private 20% · CHIP 20% · Medicare 20% · Medicaid 19% |

### Date Issues

| Column | Formats Found | Date Range | Issues |
|--------|--------------|------------|--------|
| `claim_date` | MM/DD/YYYY, YYYY-MM-DD, DD-Mon-YYYY | 2023-01-03 → 2026-12-15 | 10 future dates beyond dataset end (2025-01-31) |
| `end_date` | MM/DD/YYYY, YYYY-MM-DD, DD-Mon-YYYY | 2023-01-22 → 2025-04-17 | 4 rows where end_date is earlier than claim_date |

All three date formats appear in roughly equal thirds meaning no column can be sorted, filtered, or used in date arithmetic without standardization first.

---

## dim_patients

### Null Rates

| Column | Nulls | Null Rate | Notes |
|--------|-------|-----------|-------|
| `cust_id` | 0 | 0% | Primary key — clean |
| `full_name` | 0 | 0% | 507 distinct values across 618 rows — duplicates present |
| `email` | 102 | 16.5% | Structured missingness: ~70% of Guest-type patients have no email |
| `phone` | 46 | 7.4% | Random missingness |

### Key Findings

**Duplicate rows:** 618 rows but only 600 unique `cust_id` values:  18 exact duplicate rows injected.

**Near-duplicate names:** After normalizing case and whitespace, the 507 distinct name count drops further. Examples include `"john smith"`, `"John Smith"`, and `"John  Smith"` (double space) representing the same patient.

**Age outlier:** Maximum age value is `999` :  an invalid sentinel that inflates the mean and standard deviation. The `date_of_birth` column contains a corresponding entry from the year 1020 AD as a cascade effect.

**Structured email missingness:** ~70% of patients with `customer_type = 'Guest'` have no email. This is not random data loss :  it reflects a business rule and should be treated separately from the random nulls in other rows.

**Region mismatches:** The region column contains 7 distinct values but only 4 are valid US geographic regions. The values `Central`, `North`, and `East` are mismatched : the state and region on those rows do not align.

---

## dim_services

### Key Findings

- 206 rows but only 200 distinct `service_id` values : **6 duplicate rows** present as lowercase near-duplicates (e.g. `"lab test i"` vs `"Lab Test I"`).
- Zero nulls across all columns.
- The `active_flag` column has **8 distinct values** for what should be a binary field: `Active`, `True`, `YES`, `1` all mean true; `Inactive`, `False`, `NO`, `0` all mean false. Filtering on any single representation silently excludes the others 
---

## dim_locations

### Key Findings

- Zero nulls. All 80 `location_id` values are unique.
- **5 rows have a region that does not match the actual US region for their state**  for example, a California location labeled as "South". These cause geographic aggregations to misattribute a portion of records.

---

## dim_providers

### Key Findings

- Zero nulls. All 80 `provider_id` values are unique.
- The `active` column has the same 8-format boolean problem as `dim_services.active_flag`  values of `Active`, `True`, `YES`, and `1` all represent active status, with no single consistent format.

---

## Cross-Table: Referential Integrity

### Join Key Mismatch
The most critical structural issue in the dataset: `fact_claims` stores the patient foreign key as `customer_id`, while `dim_patients` uses `cust_id` as its primary key. A join on matching column names returns zero rows. This must be joined explicitly:

```
fact_claims.customer_id = dim_patients.cust_id
```

### Orphaned Foreign Keys

| FK Column (fact_claims) | Dimension PK | Orphaned FKs | Orphan Rate |
|-------------------------|-------------|-------------|-------------|
| `customer_id` | `dim_patients.cust_id` | ~30 | ~4% |
| `service_id` | `dim_services.service_id` | ~10 | ~3% |
| `provider_id` | `dim_providers.provider_id` | ~5 | ~3% |
| `location_id` | `dim_locations.location_id` | 0 | 0% |

An inner join on any of the first three would silently drop these rows from all aggregations. Decision: left join and flag orphans rather than discard them.

---

## Cleaning Steps Taken

After completing the audit, the following corrections were applied to produce the five `clean_` CSV files:

1. **Standardized all date columns** — unified MM/DD/YYYY, YYYY-MM-DD, and DD-Mon-YYYY to ISO 8601 (YYYY-MM-DD) across all tables
2. **Parsed all currency columns to numeric** — stripped `$` and `,` from `revenue`, `cost`, `unit_price` in both fact and dimension tables
3. **Normalized boolean columns** — standardized `active_flag`, `active`, and `primary_flag` to `0` / `1` integers
4. **Removed exact duplicate rows** — 18 from `dim_patients`, 6 from `dim_services`
5. **Resolved FK key name mismatch** — documented and handled explicitly in all joins (`customer_id` → `cust_id`)
6. **Identified and flagged orphaned FKs** — added `orphan_patient_flag`, `orphan_service_flag`, `orphan_provider_flag` columns to `fact_claims`
7. **Corrected outliers** — nulled `age > 120`, removed corresponding invalid DOBs, fixed 12 negative revenue values to their absolute value
8. **Validated and fixed business rules** — swapped 4 inverted date pairs, corrected 6 zero-quantity Completed claims, removed 10 future-dated claims
9. **Fixed region mismatches** — re-derived `region` from `state` using an authoritative state-to-region lookup in both `dim_patients` and `dim_locations`
10. **Tagged stale dimension references** — added `stale_service_flag` and `stale_provider_flag` to `fact_claims` to identify active claims linked to inactive services or providers

<img width="1140" height="656" alt="image" src="https://github.com/user-attachments/assets/750a0da8-3bba-4032-8963-64963723e93b" />
<img width="1137" height="652" alt="image" src="https://github.com/user-attachments/assets/cf5a0498-9c02-49bb-80ca-1ac67c065f1b" />





