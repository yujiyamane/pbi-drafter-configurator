---
name: pbi-drafter-configurator
description: "Analyse a CSV/Excel dataset and auto-generate a Dashboard Drafter Config Block. Reads column headers, data types, unique value counts, and value ranges to classify columns into SUM/CNT/AVG/KEY/OTHER/DATE slots. Proposes a Config Block for user approval. Does NOT generate the dashboard — only drafts the config. Use when user says: 'draft config', 'PBI config drafter', 'analyse this CSV for dashboard', 'create config from data', 'generate config block', 'config drafter', 'propose config', 'what config should I use for this data', or provides a CSV/Excel path with config drafting intent."
---

# PBI Drafter Configurator

## Overview

Analyse a CSV/Excel file and generate a Dashboard Drafter Config Block ready for user review.
**Output only**: A proposed Config Block + analysis summary. Never runs parse_config or run_factory.

Trigger phrases: `PBI config drafter`, `draft config`, `config drafter`, `analyse CSV`, `propose config`, `create config block`, `generate config from`, `what config for this data`, `make config`, `auto config`, `config for this CSV`

---

## Step 1 — Analyse the Dataset

Read the file and extract:

- All column headers (exact names, preserve case)
- Data types per column (numeric, text, date/datetime, boolean)
- Total row count
- Per column:
  - Unique value count
  - min / max / mean (numeric only)
  - First 5 unique sample values

### Column Classification Rules

Apply in this order (first match wins):

| Classification | Condition |
|---|---|
| **DATE** | dtype is date or datetime |
| **SUM** | Numeric; large value range (max > 1000 OR max–min > 1000); column name contains any of: `revenue`, `cost`, `budget`, `price`, `amount`, `salary`, `tax`, `profit`, `sales`, `income`, `expense`, `total`, `value`, `fee`, `payment`, `gross`, `net` |
| **CNT** | Numeric; integer type; high cardinality (unique count > 50% of row count OR name ends with `_id`, `Id`, `ID`, `_key`, `Key`, `_no`, `No`, `Number`); sequential pattern (max–min ≈ row count) |
| **AVG** | Numeric; small range (0–1 or 0–100); OR column name contains: `rate`, `ratio`, `margin`, `percentage`, `pct`, `score`, `index`, `avg`, `average`, `mean` |
| **DATE (categorical)** | Text type; column name matches: `month`, `quarter`, `year`, `fy`, `fiscal`, `period`, `week`, `month_name`, `monthname` — **WARN and exclude** (Date dimension provides these) |
| **KEY** | Text type; a **repeating categorical label** — a value you would group/slice/filter the measures by. Cardinality is NOT the gate. |
| **OTHER** | Text type; **near-unique free text or a per-row identifier** (long descriptions, notes, free-form strings, IDs stored as strings); or numeric that didn't match above |

### KEY vs OTHER — do NOT gate on cardinality

**The purpose of a dashboard is to answer "slice my metric BY what?"** A KEY dimension is any *repeating categorical label* the user would want to group, filter, or slice the SUM/CNT measures by. A high unique count is often a **signal of an interesting dimension** (merchant, product, customer, category), NOT a reason to demote it.

Distinguish the two by **shape of the values**, not absolute count. Compute `uniqueness_ratio = unique_count / row_count`:

| Signal | → |
|---|---|
| Short repeating label (region, category, merchant, type, status) AND `uniqueness_ratio` < ~0.6 | **KEY** |
| Long free-form text, notes, descriptions, OR `uniqueness_ratio` ≥ ~0.8 (near-unique per row) | **OTHER** |
| Per-row identifier (every row distinct) | **OTHER** (or CNT if numeric) |

### KEY priority — rank by analytical value, not cardinality

Once columns are KEY candidates, **order the Key_Dim slots by how decision-relevant each dimension is**. Ask: *"If this were my own data, what would I most want to break my numbers down by?"* Put the most important dimension in Key_Dim_1.

Domain priority hints (highest first):

| Domain | Key_Dim priority order |
|---|---|
| Personal finance / spending | category → merchant/payee → budget group → transaction type → account → direction (in/out) → provider |
| Sales / revenue | product/SKU → customer → region/territory → channel → salesperson → segment |
| Operations / logistics | location/site → product → status → carrier/vendor → priority |
| Generic fallback | the dimension a human would name first when asked "what do you want to see this broken down by?" |

### Special-case column rules

| Column pattern | Rule |
|---|---|
| **`account_number` / `acct_no` / `card_number` / IBAN-like** | Often high cardinality and not a useful slicer → lean **OTHER**, not KEY. Flag it and ask the user whether to keep as KEY, move to OTHER, or drop. |
| **Second (or later) date column** | Only **one** DATE slot exists (DateKey). The first/primary date → DateKey. Every additional date column → **OTHER**. Note which date you chose as DateKey and offer to swap. |
| **Two-value categoricals** (e.g. `credit_debit` = Credit/Debit, boolean-like flags) | Good KEY candidates. Do NOT demote just because cardinality is low. |

### Slot Limits

| Slot | Max |
|---|---|
| SUM_Measure_1..10 | 10 |
| CNT_Measure_1..5 | 5 |
| AVG_Measure_1..5 | 5 |
| Key_Dim_1..10 | 10 |
| Other_Field_1..10 | 10 |
| DateKey | 1 |
| **Total** | **41 columns max** |

If more columns than available slots: select best candidates by relevance, explain what was dropped and why.

### Ambiguous Column Handling

If a column matches two classifications equally, flag it explicitly and present both options. Ask the user to decide before finalising.

---

## Step 2 — Propose Config Block

### Format Specifier Selection

| Pattern | Specifier |
|---|---|
| Column name contains: `revenue`, `cost`, `budget`, `price`, `amount`, `salary`, `tax`, `profit`, `sales`, `income`, `expense`, `total`, `fee`, `payment`, `gross`, `net` | `($#,0.00)` |
| Integer counts (headcount, count, quantity, no decimal values) | `(#)` |
| 0–1 range OR column name contains `rate`, `margin`, `percentage`, `pct` | `(%)` |
| Numeric with 1 decimal place precision in data | `(#.0)` |
| Numeric with 2+ decimal places | `(#.00)` |
| Custom or unclear | `(#.00)` as default |

### Title — infer the dashboard's purpose from the DATA, never the filename

**Ask: "Looking at this data, what dashboard is this person trying to build?"** The title must name the *subject* and *purpose* of the analysis.

Derive it from the data's own signals, in this order:
1. **Domain** — what is this data about? Read the column names + sample values.
2. **Primary measure** — what is being measured? (the SUM/CNT/AVG columns)
3. **Primary lens** — how will it be sliced? (the top Key_Dim)
4. Combine into a human, descriptive title: `<Subject> <Measure> <Analysis-type>`.

**Hard rules:**
- Strip dates, timestamps, version numbers, and IDs from the filename.
- Keep it concise (3–6 words), human, and specific. Title Case.
- **No special characters invalid in Windows file names.** Replace `&` with `and`. Remove or substitute any forbidden character (`& < > : " / \ | ? *`).

### Output format — `/*FACTORY ... */` (EXACT — this is what pbi-drafter consumes)

```
/*FACTORY
TITLE: <inferred title>
THEME(1:nsw-blue): 1
DB(1:Oracle 2:PostgreSQL 3:Snowflake 4:CSV 5:Excel): <4 for CSV, 5 for Excel>
SOURCE: <absolute path>

1.CNT(max5): ①<csv_col> AS "Display" ②③④⑤
2.SUM(max10): ①<csv_col> AS "Display"($#,0.00) ②③④⑤⑥⑦⑧⑨⑩
3.AVG(max5): ①②③④⑤
4.DATE: <csv_col> AS "Display"
5.KEY(max10): ①<csv_col> AS "Display" ②<csv_col> AS "Display" ③④⑤⑥⑦⑧⑨⑩
6.OTHER: <csv_col> AS "Display", <csv_col> AS "Display"
*/
```

**Formatting rules (follow EXACTLY):**

1. Wrap the entire block in `/*FACTORY` (open) and `*/` (close) on their own lines.
2. Header lines use `KEY:` colon syntax (NOT `=`): `TITLE:`, `THEME(...):`, `DB(...):`, `SOURCE:`.
3. `THEME(1:nsw-blue): 1` — always default to **1** unless the user asks otherwise.
4. `DB(1:Oracle 2:PostgreSQL 3:Snowflake 4:CSV 5:Excel):` — include the full label, then the value (4=CSV, 5=Excel).
5. `SOURCE:` — absolute path exactly as provided.
6. One blank line between the SOURCE header and the numbered slot rows.
7. Slot rows numbered `1.CNT` `2.SUM` `3.AVG` `4.DATE` `5.KEY` `6.OTHER`, in that order.
8. **Circled numbers** mark slot positions: ①②③④⑤⑥⑦⑧⑨⑩.
9. **Filled slot:** `①<actual_csv_column_name> AS "Display Name"`. Use the real column name — never placeholder names.
10. **Empty/unused slot:** show just the bare circled number with nothing after it.
11. **Format specifier goes directly after the closing quote with NO space:** `AS "Amount"($#,0.00)`.
12. **DATE** is a single slot, no circled number: `4.DATE: transaction_date AS "Transaction Date"`.
13. **OTHER** is a comma-separated list (no circled numbers).
14. Display name = column header → underscores to spaces, Title Case, acronyms kept uppercase.
15. **CNT source columns become `type text` at load time** — DISTINCTCOUNT works on any type.

---

## Step 3 — Present to User

### 1. Analysis Summary Table

```
Column Analysis (N total columns, N classified):

| Column          | Type    | Unique | Range / Samples          | → Slot         | Reason                        |
|-----------------|---------|--------|--------------------------|----------------|-------------------------------|
| OrderDate       | date    | 365    | 2023-01-01 – 2023-12-31  | DateKey        | datetime dtype                |
| TotalRevenue    | numeric | 8,420  | 0 – 1,250,000            | SUM_Measure_1  | large range + "revenue" name  |
```

### 2. Warnings (if any)

```
⚠️ DATE-RELATED CATEGORICALS EXCLUDED FROM KEY:
  - MonthName (text, 12 unique) — date dimension already provides this. Excluded.

⚠️ SLOT OVERFLOW — dropped from config:
  - Description (OTHER) — high cardinality free text, low dashboard value. Dropped.

❓ AMBIGUOUS — needs your decision:
  - Score: numeric, range 0–500. Could be SUM or AVG. Which?
```

### 3. Proposed Config Block

Display the full `/*FACTORY ... */` block in a code block.

### 4. Confirmation Prompt

```
Does this look right?

You can ask me to:
- Move a column to a different slot
- Change a format specifier
- Rename a display name
- Remove a column from the config
- Change the TITLE

Once approved, use the ready-to-run prompt below.
```

### 5. Ready-to-Generate Prompt

Always append this block verbatim after the Config Block:

```
---
**Ready to generate? Run:**

Use the pbi-drafter skill.
Generate a dashboard from this config:

/*FACTORY
...(the full config block)...
*/
---
```

---

## Key Rules

- **Always output the `/*FACTORY ... */` block format.** Never the old `[Dashboard Drafter Config]` / `KEY = VALUE` style.
- **TITLE must be Windows filename-safe.** Forbidden characters: `& < > : " / \ | ? *`. Replace `&` with `and`; remove or substitute all others.
- **Use ACTUAL CSV column names before `AS`** — never the placeholder names `SUM_Measure_1`, `CNT_Measure_1`, `Key_Dim_1`, `DateKey`, `Other_Field_1`, etc.
- **Format specifier attaches with no space** after the display name.
- **Never run parse_config, run_factory, or any dashboard generation step.** This skill outputs the Config Block text only.
- Date-related categoricals (Month, Quarter, Year, FY, MonthName, Period, Week) → always warn, never assign to Key_Dim.
- If no DATE column found → leave `4.DATE:` blank, warn user that Date Intelligence features will be unavailable.
- Total columns assigned across all slots must not exceed 41.
