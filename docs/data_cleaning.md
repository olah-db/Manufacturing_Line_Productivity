# 🧹 Data Cleaning & Validation Log

How the raw production workbook was prepared for analysis, and the checks that confirm the numbers can be trusted.

**In one sentence:** the raw data was already high quality (no missing values, no duplicates), but it was laid out in a way that is hard to analyze, so the work focused on **reshaping it, handling one overnight batch, and proving the totals reconcile**.

← Back to [README](../README.md)

---

## 1. Summary

| # | Issue found | What was done | Why it matters |
|---|---|---|---|
| 1 | Downtime table has a two-row header and 12 cause columns | Removed the title row; reshaped to one row per batch and cause | Makes it possible to total, rank and filter downtime by cause |
| 2 | Blank cells in the downtime table | Treated blanks as "no downtime" (zero), not as missing data | Prevents incorrect gaps or averages |
| 3 | Batch 422148 runs past midnight (22:55 → 01:05) | Duration calculated so a "negative" time wraps across midnight | Without this, the duration would be wrong or negative |
| 4 | Batch 422148 dated 3 Sep, but it starts at 22:55 on 2 Sep | Flagged and kept as recorded | Only affects the daily rollup, not any batch-level results |
| 5 | No duration, efficiency or total-downtime fields | Added calculated columns | These are the core KPIs of the dashboard |
| 6 | Product and cause codes live in separate tables | Linked tables using keys, with every key checked | Enables operator, product and cause breakdowns |

---

## 2. Data Profile (before cleaning)

| Table | Rows | Columns | Notes |
|---|---:|---:|---|
| Line productivity | 38 | 6 | Batch-level log, 29 Aug to 3 Sep 2024 |
| Line downtime | 38 | 13 | Wide layout: one column per cause (1–12) |
| Products | 6 | 4 | Five 600 ml products and one 2 L product |
| Downtime factors | 12 | 3 | Cause descriptions and operator-error flag |

**Initial quality scan**

| Check | Result |
|---|---|
| Missing values in the batch log | 0 |
| Duplicate batch IDs | 0 |
| Extra spaces or inconsistent spelling in product and operator names | None found |
| Batch IDs are sequential (422111 → 422148) | Yes |
| Negative downtime values | 0 |

---

## 3. Cleaning Steps in Detail

### Step 1: Fix the downtime table layout
**Problem.** The downtime sheet has a merged title ("Downtime factor") above the real column headers, and it spreads the 12 causes across 12 columns. That is easy to read by eye but awkward to filter or chart.

**Action.**
- Removed the title row so the real headers (Batch, 1, 2, … 12) sit on row 1
- Reshaped the table from **wide** (38 rows × 12 cause columns) to **long** (one row per batch and cause), using Power Query's *Unpivot Other Columns*
- Kept only rows where downtime was actually logged

**Result.** 456 possible batch-and-cause cells became **61 downtime events**, averaging 22.8 minutes each (range: 5 to 60 minutes).

### Step 2: Interpret blank cells
**Problem.** 395 of 456 cells in the downtime table are empty.

**Decision.** A blank means *no downtime was recorded for that cause*, so it is treated as **zero minutes**, not as missing data. This is supported by the reconciliation in Section 4, where logged downtime matches lost time exactly for every batch. If blanks had hidden unrecorded problems, the totals would not match.

### Step 3: Handle the overnight batch
**Problem.** Batch 422148 starts at 22:55 and ends at 01:05 the next day. Excel stores times with no date, so the end time looks *earlier* than the start time, and a simple subtraction gives a negative duration.

**Action.** Duration is calculated so that times wrap around midnight:

```excel
Duration (min) = MOD(End Time − Start Time, 1) × 1440
```

**Result.** Batch 422148 correctly shows **130 minutes** (22:55 to 01:05).

### Step 4: Flag a date inconsistency
**Observation.** Batch 422148 begins at 22:55, the exact minute the previous batch (422147, dated 2 Sep) finished, yet it is dated **3 Sep**. It most likely started on 2 Sep and was dated by the day it ended.

**Decision.** Kept as recorded, because the source system is the authority and this is an inference. **Impact is limited:** it only shifts one batch between 2 Sep and 3 Sep in any *daily* view. All batch-, operator-, product- and cause-level results are unaffected.

### Step 5: Add calculated fields

| Field | Formula | Purpose |
|---|---|---|
| Duration (min) | `MOD(End − Start, 1) × 1440` | Actual time taken |
| Min batch time | Lookup from Products by Product ID | Ideal time with no downtime |
| Total downtime (min) | Sum of logged minutes per batch | Time lost |
| Excess time (min) | Duration − Min batch time | Used to validate downtime |
| Line efficiency | Min batch time ÷ Duration | Headline KPI |
| Operator-error downtime | Downtime from causes flagged "Yes" | Controllable losses |

### Step 6: Link the tables
Tables were connected through three keys:

- `Line productivity.Product` → `Products.Product`
- `Line productivity.Batch` → `Line downtime.Batch`
- `Line downtime` cause → `Downtime factors.Factor`

---

## 4. Validation Checks

| Check | Result |
|---|---|
| Every batch in the log has a downtime row, and vice versa | ✅ Pass (38 of 38 match) |
| Every product ID exists in the Products table | ✅ Pass |
| Every downtime cause (1–12) exists in the Downtime factors table | ✅ Pass |
| No batch finishes faster than its minimum time | ✅ Pass (0 of 38) |
| **Logged downtime = actual time − minimum time, per batch** | ✅ **Pass (38 of 38, zero difference)** |
| Overall totals tie out | ✅ 3,858 actual − 2,470 minimum = **1,388 min = total logged downtime** |
| Cause-level totals add up to overall downtime | ✅ Operator error 776 + other 612 = 1,388 |
| Batches are back to back within each production day | ✅ 34 of 37 handovers are seamless; the 3 breaks fall between production days |

> **Why the reconciliation matters:** every minute lost was explained by a logged cause, and nothing was unexplained or double counted. That gives high confidence in the cause rankings and efficiency figures.

---

## 5. Assumptions & Judgment Calls

1. **Blank downtime cells equal zero minutes**, supported by the reconciliation above.
2. **"Operator Error" flags are used exactly as provided** in the Downtime factors table. The classification was not audited or changed.
3. **Minimum batch time is a fixed standard** per product (60 min for 600 ml, 98 min for 2 L) and treated as the ideal benchmark.
4. **The 3 Sep date on batch 422148 is kept as recorded** (see Step 4).
5. **No rows were removed or altered.** All 38 batches and 61 events are included in the analysis.

---

## 6. Known Limitations

- **Sample size:** 38 batches over 5 days, so results are directional.
- **One product (OR-600) has a single batch,** so its metrics are not representative.
- **Downtime is recorded in whole minutes** by cause, so no detail is available on *when within a batch* each stoppage occurred.

---

## 7. Reproducing the Cleaning

1. Open `Manufacturing_Line_Productivity.xlsx` and load each sheet into Power Query (**Data → Get & Transform**).
2. On *Line downtime*: remove the top title row, promote the next row to headers, select the *Batch* column, then **Unpivot Other Columns** and remove nulls.
3. On *Line productivity*: add the duration column using the formula in Step 5.
4. Merge in *Products* and *Downtime factors* to bring in minimum batch time, cause description and operator-error flag.
5. Run the validation checks in Section 4 and confirm all pass.
