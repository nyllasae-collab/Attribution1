# Rebate Classification Query


A production PostgreSQL query that evaluates purchase-level data across multiple tables to determine rebate type, calculate rebate amount, and surface supporting fields for downstream visualization.

## Background

Rebate eligibility and calculation in a food distribution context involves layered business logic — contract types, attribution tracking, product normalization, and vendor program rules all intersect at the transaction level. This query was written to handle that complexity in a single pass, feeding a reporting layer that reads directly from the output.

## What It Does

- Joins purchase attribution data against contract master records using a **lateral join with priority ordering** — ensuring the correct contract type is applied when multiple matches exist
- Calculates rebate amounts dynamically across three calculation types:
  - `FIXED_AMOUNT_PER_WEIGHT` — normalized to ounces, multiplied by quantity
  - `FIXED_AMOUNT_PER_QTY` — flat rate per unit purchased
  - `PERCENT_OF_PRICE` — percentage applied to invoice line total
- Uses `REGEXP_REPLACE` to sanitize raw calculation amount strings before casting to numeric
- Resolves missing attribution data via a **deduplication CTE** (`origattribdata`) that back-fills original product identifiers from prior attribution records
- Applies `COALESCE` fallback logic throughout to prefer transaction-level overrides before falling back to contract master defaults
- Filters out invalid attribution/contract combinations via explicit exclusion logic

## Notable Techniques

| Technique | Purpose |
|---|---|
| Lateral join with `ORDER BY` + `LIMIT 1` | Priority-based contract matching without window functions |
| CTE deduplication | Back-fill original product data from attribution history |
| `REGEXP_REPLACE` inside `CAST` | Normalize dirty numeric strings from source data |
| `COALESCE` on calculation fields | Transaction-level overrides with contract master fallback |
| Multi-condition `WHERE` exclusions | Filter invalid contract/attribution type combinations |

## Usage

This query is designed to run against a PostgreSQL database. The primary output feeds a visualization layer reading from `table_a`. No modifications are needed for standard use — to adapt for a different client or date range, update the filter conditions in the `WHERE` clause.

```sql
-- Key filter conditions to adjust as needed:
where gd.client = 'CLIENT_A'
and ("attribClaimedSticker" is not null OR "priorAttribSticker" is not null)
```

## Schema Dependencies

| Table | Role |
|---|---|
| `attribution` | Core purchase + attribution records |
| `account_detail` | Account-to-group mapping |
| `group_detail` | Group and aggregator metadata |
| `contract_master` | Rebate program definitions |
| `purchase_history` | Raw purchase records |
