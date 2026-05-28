# Bright Coffee Shop Sales Performance
### SQL Portfolio Project: Pattern Matching & Window Functions

---

## 📌 Project Overview
This project focuses on analyzing transactional sales data from **Bright Coffee Shop**, a chain with three brick-and-mortar locations:
*   Lower Manhattan
*   Hell's Kitchen
*   Astoria

The goal of this analysis is to clean up unstructured product descriptions, track business revenue distributions, calculate running totals, and run localized row rankings using advanced SQL techniques in Databricks.

---

## 🗃️ Dataset Architecture
The core queries are written in **Databricks SQL** against the `coffee_sales` ledger table.

| Column Name | Data Type | Description |
| :--- | :--- | :--- |
| `transaction_id` | INT | Unique ID for each customer checkout. |
| `transaction_date` | DATE | Date the sale occurred. |
| `transaction_time` | TIME | Time the sale occurred. |
| `transaction_qty` | INT | Number of items purchased. |
| `store_id` | INT | Unique identifier for the store location. |
| `store_location` | VARCHAR | Store neighborhood (Astoria, Hell's Kitchen, Lower Manhattan). |
| `product_id` | INT | Unique identifier for the item. |
| `unit_price` | DECIMAL | Price per single unit. |
| `product_category`| VARCHAR | Item category (e.g., Coffee, Bakery, Tea). |
| `product_type` | VARCHAR | Specific item sub-type (e.g., Chai). |
| `product_detail` | VARCHAR | Detailed item menu name, including size codes (Sm, Rg, Lg). |

> **Calculated Field:** 
> Individual transaction revenue is derived dynamically using:
> `Revenue = transaction_qty * unit_price`

---

## 🛠️ Key SQL Implementations

### 1. Pattern Matching & Text Filtering (Wildcards)
Used standard text matching to isolate and filter specific inventory subsets based on product naming formats:
*   **Prefixes (`LIKE 'Dark%'`)**: Finds products starting with specific words, like dark roasts.
*   **Suffixes (`LIKE '%Lg'`)**: Extracts specific sizing tiers from the end of detailed strings.
*   **Substrings (`LIKE '%Chai%'`)**: Checks for keywords anywhere in a text string.
*   **Exclusions (`NOT LIKE '%Tea%'`)**: Cleanses reports by removing entire product segments.
*   **Fixed Character Alignment (`_`)**: Uses single-character wildcards to pinpoint exact layout patterns, such as finding text ending with specific space-separated size blocks (`' % Rg'`).
*   **Regex Operations (`RLIKE '[0-9]'`)**: Implements regular expression constraints to find rows that contain numbers or specific alphanumeric characters.

### 2. Advanced Window Functions
Applied partitioning and sorting frames to run calculations across blocks of related data without collapsing separate rows.

#### Aggregate Functions (`SUM`, `AVG`)
*   **Static Window Partitioning:** Isolating blocks with `PARTITION BY` alone calculates an overall total across an entire store partition. This allows us to find the exact percentage of total store revenue that a single transaction represents.
*   **Cumulative Running Totals:** Adding an `ORDER BY` clause inside the window shifts the calculation to a dynamic rolling frame, tracking revenue chronologically as it builds up transaction-by-transaction.

#### Ranking Methods (`ROW_NUMBER`, `RANK`, `DENSE_RANK`)
Handled identical price points and structural data ties by testing different ranking sequences:
*   `ROW_NUMBER()` outputs a continuous order sequence (`1, 2, 3, 4`), ignoring ties.
*   `RANK()` handles ties with identical indexes but leaves gaps right after (`1, 1, 1, 4`).
*   `DENSE_RANK()` groups identical rows under a shared index and continues immediately with the next number (`1, 1, 1, 2`).

#### Value Offsets (`LAG`, `LEAD`, `FIRST_VALUE`)
*   **Pacing Tracking:** Used `LAG()` and `LEAD()` to look backward or forward by an offset, letting us compare adjacent transactions chronologically without heavy self-joins.
*   **Boundary States:** Handled edge cases where the last row of a store partition natively yields a `NULL` value on subsequent row lookups.
*   **Baseline Referencing:** Used `FIRST_VALUE()` to isolate the lowest price item within a product line and display it directly next to all active items in that group.

---

## 🏆 Final Synthesis Query
This comprehensive script combines text pattern wildcards with nested row-ranking and window averages into a single workflow:

```sql
WITH filtered_sales AS (
    -- Step 1: Filter down to specific regular and large sizes using wildcards
    SELECT transaction_id, store_location, product_detail, transaction_qty, unit_price,
           (transaction_qty * unit_price) AS revenue
    FROM coffee_sales
    WHERE product_detail LIKE '%Lg'
       OR product_detail LIKE '%Rg'
)
-- Step 2: Calculate local store rankings and averages concurrently
SELECT transaction_id, store_location, product_detail,
       ROUND(revenue, 2) AS revenue,
       RANK() OVER(
           PARTITION BY store_location 
           ORDER BY revenue DESC
       ) AS store_rank,
       ROUND(AVG(revenue) OVER(
           PARTITION BY store_location
       ), 2) AS store_avg_revenue
FROM filtered_sales
ORDER BY store_location, store_rank
LIMIT 40;
