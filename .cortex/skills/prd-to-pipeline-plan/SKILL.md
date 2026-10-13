---
name: prd-to-pipeline-plan
description: >
  Reads a PRD-style XLSX file (source onboarding requests, business rules, field mappings)
  and produces a structured implementation plan for updating a target Dynamic Table.
  Use when: new PRD arrives, business requirements change, source onboarding review,
  pipeline impact analysis, planning Silver layer changes.
  Triggers: PRD, business requirements, pipeline plan, source onboarding, impact analysis,
  new source system, Silver layer changes, requirement review, change summary.
---

# PRD to Pipeline Plan

Reads product/business requirement documents (XLSX) and produces a structured
implementation plan for a target Snowflake Dynamic Table in a medallion architecture.

## Inputs

Collect these from the user before starting analysis. Do not guess defaults.

| Input | Required | Description |
|---|---|---|
| `prd_path` | Yes | Absolute path to the XLSX file containing PRD sheets (source onboarding, business rules, field mappings, etc.) |
| `target_dynamic_table` | Yes | Name of the Silver-layer Dynamic Table being changed (e.g., `SILVER_AP_INVOICES`) |

If the user provides a directory instead of a file, list XLSX files in that directory
and ask them to pick one. If the file has multiple sheets, read all sheets and use
sheet names to classify content (see Step 2).

## Workflow

### Step 1: Confirm Inputs

Ask the user for `prd_path` and `target_dynamic_table` if not already provided.
Verify the file exists and is readable.

**STOP**: Confirm both inputs with the user before proceeding.

### Step 2: Read and Classify PRD Sheets

Use `read_tabular` to load each sheet in the XLSX file. Classify each sheet by
inspecting column headers:

| Sheet Type | Identifying Columns | What to Extract |
|---|---|---|
| Source Onboarding | `Source System`, `ERP Platform`, `Region`, `Go-Live Target`, `Status` | New source systems, platforms, regions, timelines, blockers |
| Business Rules | `Rule ID`, `Category`, `Rule Description`, `Decision` | Transformation logic, normalization rules, data quality rules |
| Field Mappings | `Source Field`, `Target Field`, `Transformation` | Column mappings, type conversions, rename rules |
| Data Quality | `Check`, `Threshold`, `Action` | DQ expectations, alerting rules, threshold values |

If a sheet does not match any known type, report it as unclassified and show its
columns to the user. Do not skip unclassified sheets — they may contain relevant
context.

Read every sheet. Do not sample or truncate. The PRD is the system of record for
this analysis.

**STOP**: Show the user which sheets were found, how each was classified, and the
row counts. Confirm interpretation before analyzing.

### Step 3: Extract New Source Systems

From source-onboarding sheets, identify each source system entry and extract:

- Source system name and ERP platform
- Region and deployment model (cloud, on-prem, hybrid)
- Data delivery mechanism (CDC, batch, API, file drop) and timing
- Go-live target and current status
- Blocking dependencies (legal, technical, access)
- Contact/owner information

**For each new source, check whether it already exists in the target DT.** If the
user's project contains SQL or dbt models for the target DT, read them to determine
which SOURCE_SYSTEM values are already handled. Report which sources are net-new
vs. already integrated.

### Step 4: Extract Field and Rule Changes for the Silver Layer

From business-rules and field-mapping sheets, extract every rule that affects the
Silver layer. Organize into:

**Transformation Rules** — status normalization, type conversions, field renames,
CASE statements, format standardization.

**Structural Rules** — new columns to add, columns to drop at bronze-to-silver
boundary, columns to carry through unchanged.

**Data Quality Rules** — thresholds, flags, dedup logic, validation checks.
Distinguish between rules that belong IN the DT definition vs. rules that should
be implemented as external checks (DMFs, alerts, tasks).

**Cross-Reference Rules** — joins to reference tables, lookups, enrichments.
Note whether the reference data is in-scope or deferred.

**Refresh/Timing Rules** — TARGET_LAG settings, source arrival windows, ordering
dependencies between sources.

For each rule, capture:
- Rule ID (if provided in the PRD)
- What it does (plain language)
- Which source systems it applies to
- Where it should be implemented (Bronze, Silver DT, Gold, external DQ)
- Decision status (confirmed, pending, open question)

### Step 5: Surface Ambiguities and Open Questions

This is the most important step. Scan every extracted item for:

**Explicitly marked open questions** — anything the PRD flags as "OPEN QUESTION",
"NEEDS DECISION", "TBD", "pending", or similar.

**Implicit ambiguities** — problems the PRD does not flag but that an engineer
would need resolved:
- Rules that reference data outside the project scope (e.g., FX rate tables,
  chart-of-accounts mappings) without confirming availability
- Format changes or historical data inconsistencies mentioned in notes but not
  addressed by a rule
- Threshold-based rules that depend on currency conversion when Silver stores
  original currencies
- Status or category values mentioned only in notes/comments, not in the
  normalization mapping
- Scope conflicts (e.g., a source's region doesn't match the data it sends)
- Rules where the PRD says "normalize" but doesn't specify which layer
- Legal or contractual dependencies with no confirmed resolution date

**Unstated assumptions** — things the plan must assume to proceed:
- That reference tables exist and are populated
- That source system extracts include all required fields
- That historical data follows the same format as current data
- That batch timing windows won't overlap or conflict

**DO NOT resolve ambiguities by guessing.** Report each one with:
- What the ambiguity is
- Why it matters for implementation
- What decision or information is needed
- Who owns the decision (if the PRD says)

### Step 6: Produce Output

Return ALL of the following sections. Do not omit any section even if it would be
empty — state "None identified" instead.

---

#### 1. New Source Systems

For each new source:
- Name, platform, region
- Delivery mechanism and timing
- Status and blockers
- Net-new vs. already integrated

#### 2. Silver Layer Field Changes

Table of fields being added, modified, or dropped:
| Field | Change Type | Source Systems | Rule ID | Notes |
|---|---|---|---|---|

#### 3. Silver Layer Business Rules

For each rule affecting the target DT:
- Rule ID, category, plain-language description
- Implementation location (Silver DT / Bronze / Gold / external DQ)
- Decision status (confirmed / pending / open)
- SQL sketch or pseudocode if the PRD specifies logic

#### 4. Ambiguities and Open Questions

Numbered list. For each:
- The question
- Why it blocks or risks implementation
- Who owns the decision
- Recommended default if the team needs to proceed before resolution

#### 5. Implementation Checklist

Ordered list of concrete steps to implement the changes, suitable for turning
into tasks or tickets:
- [ ] Step description — references rule IDs and source systems

---

**STOP**: Present the full output to the user for review. Ask if any section
needs correction or deeper analysis.

## Rules

- **Never guess.** If the PRD is ambiguous, say so in Section 4. Do not fill gaps
  with assumptions presented as facts.
- **Read every sheet.** Do not skip sheets even if their names seem irrelevant.
- **Preserve Rule IDs.** Always reference the PRD's own identifiers (BR-001, SRC-2025-003, etc.).
- **Distinguish DT-internal vs. external.** Not everything belongs in the Silver DT.
  Data quality checks, alerting, and cross-reference joins may belong elsewhere.
- **Show your sources.** When citing a rule or requirement, reference the sheet name
  and row/Rule ID so the user can trace back to the PRD.
- **One target at a time.** If the PRD covers multiple pipelines, ask the user which
  target DT to focus on. Do not mix concerns across targets.

## Example Usage

**User:**
> I have a new PRD at `./requirements/ap_invoices_q3.xlsx`. Analyze it for changes
> to `SILVER_AP_INVOICES`.

**Skill produces:**

### 1. New Source Systems
| Source | Platform | Region | Delivery | Status |
|---|---|---|---|---|
| Baan IV | Infor Baan | EMEA | Nightly CSV to S3 ~02:00 UTC | Approved, mapping in progress |
| Workday FM | Workday | Americas | Hourly via connector | Pending legal review (DPA-2025-0041) |

### 2. Silver Layer Field Changes
| Field | Change | Sources | Rule |
|---|---|---|---|
| SOURCE_SYSTEM | Add literal `'BAAN'`, `'WORKDAY'` in UNION ALL | Baan, Workday | BR-007 |
| BAN_COMPANY | Drop at bronze-to-silver | Baan | BR-008 |
| WD_TENANT_ID | Drop at bronze-to-silver | Workday | BR-008 |
| STATUS | Add CASE branches: `POSTED`->`APPROVED`, `Approved`->`APPROVED`, `In Review`->`PENDING` | Baan, Workday | BR-001 |

### 3. Silver Layer Business Rules
- **BR-003** (Dedup): `QUALIFY ROW_NUMBER() OVER (PARTITION BY INVOICE_NUMBER ORDER BY CREATED_AT DESC) = 1` — Silver DT, confirmed.
- **BR-004** (Amount threshold): Flag >$500K — implement as DMF, NOT in Silver DT. Confirmed.
- **BR-002** (Currency): No conversion at Silver, store original. Confirmed.

### 4. Ambiguities and Open Questions
1. **Payment terms normalization (BR-005)**: Silver or Gold? Decision pending (Sarah Chen / David Kim). *Recommend: leave raw at Silver, normalize at Gold.*
2. **Baan cost center format change**: Old `BC-XX` vs new `BC-XXX` in historical data. No rule covers this. *Need: decision on passthrough vs. standardization.*
3. **$500K threshold uses USD equivalent (BR-004)**: Silver has no FX conversion. *May only be enforceable at Gold layer.*
4. **Workday tenant WD-T2 scope**: PRD says Americas but WD-T2 is London (GBP/EUR). *Need: confirm in-scope or out-of-scope.*

### 5. Implementation Checklist
- [ ] Add Baan bronze source table and ingestion (nightly S3 CSV)
- [ ] Add Workday bronze source table and ingestion (hourly connector)
- [ ] Extend Silver DT UNION ALL with Baan branch (SOURCE_SYSTEM='BAAN', status mapping, BAN_COMPANY drop, dedup logic)
- [ ] Extend Silver DT UNION ALL with Workday branch (SOURCE_SYSTEM='WORKDAY', status mapping, WD_TENANT_ID drop)
- [ ] Resolve BR-005 (payment terms layer) before Gold implementation
- [ ] Confirm Baan cost center handling with Karen van der Berg
- [ ] Confirm Workday WD-T2 scope with Jennifer Okafor
- [ ] Add BR-004 DMF for high-value invoice alerting
- [ ] Validate with 30-day monitoring window per BR-009
