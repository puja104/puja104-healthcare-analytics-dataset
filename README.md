# Data Audit & Profiling Report

## Overview

I first conducted a data profiling audit on six tables using Power Query. The analysis examined null values, distinct value counts, any promblems with the data. 

## Table: fact_claims

| Column | Nulls | Null Rate |
|--------|-------|-----------|
| `claim_date` | 37 | **4.8%** | 
| `revenue` | 52 | **6.7%** | 
| `cost` | 51 | **6.6%** | 
| `copay` | 88 | **11.3%** | 
| `satisfaction_score` | 119 | **15.3%** | 

### Numeric Distributions

| Column | Min | Max | Mean | Median | Std Dev | p99 | Negatives | Zeros |
|--------|-----|-----|------|--------|---------|-----|-----------|-------|
| `quantity` | 0 | 10 | 5.29 | 5.0 | 2.86 | 10 | 0 | **12** |
| `unit_price` | $52 | $4,982 | $2,546 | $2,611 | $1,391 | $4,943 | 0 | 0 |
| `revenue` | **-$35,319**  | $809,797 | $18,881 | $10,645 | $48,490 | $257,849 | **13** | **2**  |
| `cost` | $70 | $33,151 | $6,594 | $4,932 | $5,749 | $25,868 | 0 | 0 |
| `copay` | $0.71 | $499.99 | $252.58 | $258.27 | $145.50 | $498.15 | 0 | 0 |
| `satisfaction_score` | 1 | 10 | 5.42 | 5.0 | 2.89 | 10 | 0 | 0 |


## dim_patients 

### Null Rates
| Column | Nulls | Notes |
|--------|------|-------|
| `cust_id` | 0 | PK |
| `email` | 102 | structured missingness (Guest users) |
| `phone` | 46 | random missing |
| `full_name` | 0 | 507 distinct values → duplicates present |

- Duplicate IDs: 18 duplicate rows
- Age outlier: max = 999 (invalid sentinel)
- Region mismatches: invalid values (`Central`, `North`, `East`)
- Email missingness is structured, not random

 ## dim_services
- 0 nulls
- 200 distinct service_id values
- 6 duplicate service rows

 ## dim_locations 
Null Rates
- 0 nulls. All 80 rows are unique.
- 5 location rows have a region that doesn't match the actual US region for their state.

## dim_providers
Null Rates
- All columns: 0 nulls. All 80 provider_id values are unique.
- multi-format boolean problem as dim_services.active_flag

## Notes
1. Standardize all date columns
2. Parse currency columns to numeric 
3. Normalize boolean columns 
4. Remove exact duplicates 
5. Resolve FK key name mismatch 
6. Identify and tag orphaned FKs
7. Flag and investigate outliers
8. Validate business rules 
9. Fix region mismatches
10. Tag stale dimension references 
