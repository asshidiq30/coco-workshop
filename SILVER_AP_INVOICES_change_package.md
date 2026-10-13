# SILVER_AP_INVOICES — PRD Change Package

> Generated from PRD analysis session. Another engineer can use this file to
> review the planned change, rerun validation, and invoke the skill on future PRDs.

## Artifact Inventory

| # | Artifact | Path | Purpose |
|---|---|---|---|
| 1 | Source onboarding requests | `sample_business_requirements_source_onboarding.csv` | New source systems (Baan IV, Workday FM) — owners, timelines, blockers |
| 2 | Business rules | `sample_business_requirements_business_rules.csv` | BR-001 through BR-010 — status normalization, dedup, currency, DQ, retention |
| 3 | Column mapping | `sample_business_requirements_column_mapping.csv` | Field-level source-to-Silver mapping for Baan (16 fields) and Workday (16 fields) |
| 4 | PRD Evaluator skill | `.cortex/skills/prd-to-pipeline-plan/SKILL.md` | Reusable skill — invoke with `/prd-to-pipeline-plan` on any future PRD |

## How to Rerun This Analysis

```
/prd-to-pipeline-plan

Inputs:
  prd_path: sample_business_requirements_column_mapping.csv
  target_dynamic_table: SILVER_AP_INVOICES
```

Cross-reference the companion files (artifacts 1 and 2) for source onboarding
context and business rule decisions.

## Open Questions (must resolve before merge)

| # | Question | Owner | Recommended Default |
|---|---|---|---|
| 1 | Payment terms normalization — Silver or Gold? (BR-005) | Sarah Chen / David Kim | Leave raw at Silver, normalize at Gold |
| 2 | Baan cost center format change — `BC-XX` vs `BC-XXX` in historical data | Karen van der Berg | Pass through as-is at Silver |
| 3 | BR-004 $500K threshold uses "USD equivalent" but Silver has no FX conversion | Tom Walsh | Implement DMF at Gold after FX conversion |
| 4 | Workday tenant WD-T2 (London, GBP/EUR) — in-scope or out-of-scope? | Jennifer Okafor | Assume in-scope, confirm with Jennifer |
| 5 | Workday legal blocker (DPA-2025-0041) — not yet signed | Jennifer Okafor / Legal | Build branch now, gate go-live on signoff |

## DDL Delta — New UNION ALL Branches

### Baan IV (with BR-003 dedup)

```sql
SELECT
    BAN_INVOICE_ID          AS INVOICE_ID,
    BAN_INVOICE_REF         AS INVOICE_NUMBER,
    BAN_VENDOR_CODE         AS VENDOR_ID,
    BAN_VENDOR_DESC         AS VENDOR_NAME,
    BAN_INV_DATE            AS INVOICE_DATE,
    BAN_PAY_DATE            AS DUE_DATE,
    BAN_AMOUNT              AS INVOICE_AMOUNT,
    BAN_CURR                AS CURRENCY_CODE,
    BAN_PAY_TERMS           AS PAYMENT_TERMS,
    BAN_PO_REF              AS PO_NUMBER,
    BAN_LINE_DESC           AS LINE_DESCRIPTION,
    BAN_GL_CODE             AS GL_ACCOUNT,
    BAN_COST_CTR            AS COST_CENTER,
    CASE BAN_STATUS
        WHEN 'POSTED'   THEN 'APPROVED'
        WHEN 'APPROVED' THEN 'APPROVED'
        WHEN 'PENDING'  THEN 'PENDING'
        ELSE BAN_STATUS
    END                     AS APPROVAL_STATUS,
    BAN_CREATED             AS CREATED_AT,
    'BAAN'                  AS SOURCE_SYSTEM
FROM BRONZE_BAAN_AP_INVOICES
QUALIFY ROW_NUMBER() OVER (
    PARTITION BY BAN_INVOICE_REF
    ORDER BY BAN_CREATED DESC
) = 1
```

### Workday

```sql
SELECT
    WD_INVOICE_ID           AS INVOICE_ID,
    WD_INVOICE_NUM          AS INVOICE_NUMBER,
    WD_SUPPLIER_ID          AS VENDOR_ID,
    WD_SUPPLIER_NAME        AS VENDOR_NAME,
    WD_INVOICE_DATE         AS INVOICE_DATE,
    WD_DUE_DATE             AS DUE_DATE,
    WD_AMOUNT               AS INVOICE_AMOUNT,
    WD_CURRENCY             AS CURRENCY_CODE,
    WD_PAY_TERMS            AS PAYMENT_TERMS,
    WD_PO_NUMBER            AS PO_NUMBER,
    WD_MEMO                 AS LINE_DESCRIPTION,
    WD_LEDGER_ACCOUNT       AS GL_ACCOUNT,
    WD_COST_CENTER          AS COST_CENTER,
    CASE WD_APPROVAL_STATUS
        WHEN 'Approved'  THEN 'APPROVED'
        WHEN 'In Review' THEN 'PENDING'
        ELSE WD_APPROVAL_STATUS
    END                     AS APPROVAL_STATUS,
    WD_CREATED_DATE         AS CREATED_AT,
    'WORKDAY'               AS SOURCE_SYSTEM
FROM BRONZE_WORKDAY_AP_INVOICES
```

## Post-Implementation Validation Queries

### 1. Row count by source

```sql
SELECT SOURCE_SYSTEM, COUNT(*) AS ROW_COUNT
FROM SILVER_AP_INVOICES
GROUP BY SOURCE_SYSTEM
ORDER BY SOURCE_SYSTEM;
```

### 2. Baan dedup — no duplicate INVOICE_NUMBERs (BR-003)

```sql
SELECT INVOICE_NUMBER, COUNT(*) AS CNT
FROM SILVER_AP_INVOICES
WHERE SOURCE_SYSTEM = 'BAAN'
GROUP BY INVOICE_NUMBER
HAVING CNT > 1;
-- Expected: 0 rows
```

### 3. Status normalization — no unmapped values (BR-001)

```sql
SELECT SOURCE_SYSTEM, APPROVAL_STATUS, COUNT(*) AS CNT
FROM SILVER_AP_INVOICES
WHERE APPROVAL_STATUS NOT IN ('APPROVED', 'PENDING')
GROUP BY SOURCE_SYSTEM, APPROVAL_STATUS;
-- Expected: 0 rows
```

### 4. Dropped columns — confirm absent (BR-008)

```sql
SELECT * FROM SILVER_AP_INVOICES LIMIT 0;
-- BAN_COMPANY and WD_TENANT_ID must NOT appear
```

### 5. NULL check on required fields

```sql
SELECT
    SOURCE_SYSTEM,
    COUNT_IF(INVOICE_ID IS NULL)       AS NULL_INVOICE_ID,
    COUNT_IF(INVOICE_NUMBER IS NULL)   AS NULL_INVOICE_NUMBER,
    COUNT_IF(VENDOR_ID IS NULL)        AS NULL_VENDOR_ID,
    COUNT_IF(INVOICE_DATE IS NULL)     AS NULL_INVOICE_DATE,
    COUNT_IF(INVOICE_AMOUNT IS NULL)   AS NULL_AMOUNT,
    COUNT_IF(CURRENCY_CODE IS NULL)    AS NULL_CURRENCY,
    COUNT_IF(GL_ACCOUNT IS NULL)       AS NULL_GL,
    COUNT_IF(APPROVAL_STATUS IS NULL)  AS NULL_STATUS,
    COUNT_IF(CREATED_AT IS NULL)       AS NULL_CREATED
FROM SILVER_AP_INVOICES
WHERE SOURCE_SYSTEM IN ('BAAN', 'WORKDAY')
GROUP BY SOURCE_SYSTEM;
-- Expected: all zeros
```

### 6. Currency code distribution (BR-002)

```sql
SELECT SOURCE_SYSTEM, CURRENCY_CODE, COUNT(*) AS CNT
FROM SILVER_AP_INVOICES
WHERE SOURCE_SYSTEM IN ('BAAN', 'WORKDAY')
GROUP BY SOURCE_SYSTEM, CURRENCY_CODE
ORDER BY SOURCE_SYSTEM, CNT DESC;
-- Expected: Baan=EUR,GBP. Workday=USD,GBP,EUR.
```

### 7. Baan cost center format audit (open question #2)

```sql
SELECT
    COST_CENTER,
    CASE
        WHEN COST_CENTER RLIKE 'BC-[0-9]{2}' THEN 'OLD (BC-XX)'
        WHEN COST_CENTER RLIKE 'BC-[0-9]{3}' THEN 'NEW (BC-XXX)'
        ELSE 'UNEXPECTED'
    END AS FORMAT_TYPE,
    COUNT(*) AS CNT
FROM SILVER_AP_INVOICES
WHERE SOURCE_SYSTEM = 'BAAN'
GROUP BY COST_CENTER, FORMAT_TYPE
ORDER BY FORMAT_TYPE, COST_CENTER;
```

## Rule Reference

| Rule | Category | Layer | Status |
|---|---|---|---|
| BR-001 | Status normalization | Silver DT (CASE) | Confirmed |
| BR-002 | Currency — no conversion | Silver (passthrough) | Confirmed |
| BR-003 | Baan dedup on INVOICE_NUMBER | Silver DT (QUALIFY) | Confirmed |
| BR-004 | Amount >$500K flag | External DMF | Confirmed (but see open question #3) |
| BR-005 | Payment terms normalization | **Open — Silver or Gold?** | Pending decision |
| BR-006 | GL account — no cross-map | Silver (passthrough) | Confirmed |
| BR-007 | SOURCE_SYSTEM literal | Silver DT | Confirmed |
| BR-008 | Drop system-specific columns | Silver DT | Confirmed |
| BR-009 | Refresh timing / TARGET_LAG | DT config | Confirmed |
| BR-010 | Data retention | Phase 2 | Noted |
