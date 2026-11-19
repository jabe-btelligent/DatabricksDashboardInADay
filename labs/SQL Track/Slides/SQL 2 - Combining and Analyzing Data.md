# 🧮 SQL 2 – Combining and Analyzing Data

**SQL Track · Presentation for the hands-on lab**

| Today | Concepts |
|---|---|
| 🥞 Stack and compare result sets | `UNION ALL`, `UNION`, `MINUS`, `INTERSECT` |
| 🧱 Write readable queries | `WITH … AS` (CTEs) |
| 🪟 Look at neighbouring rows | window functions with `PARTITION BY` |
| ➕ Subtotals for reports | `ROLLUP`, `CUBE`, `GROUPING()` |
| 🌳 Hierarchies | `WITH RECURSIVE` |

---

## 🧭 Joins vs. set operators

| | Joins (SQL 1) | Set operators (today) |
|---|---|---|
| combine … | **columns** – side by side | **rows** – on top of each other |
| match rows by … | a key in the `ON` clause | **all** columns, by position |
| example | order line + product name | café orders + online orders |

```text
JOIN:   [ order | product ]          UNION:   [ café orders   ]
                                              [ online orders ]
```

Rules for set operators: both queries need the **same number of columns** with **compatible types**. Columns are matched **by position**, the names come from the **first** query.

---

## 🥞 UNION ALL vs. UNION

```sql
SELECT city FROM VALUES ('Hamburg'), ('Berlin') AS cafe(city)
UNION ALL
SELECT city FROM VALUES ('Berlin'), ('Leipzig') AS online(city);
```

| `UNION ALL` | `UNION` |
|---|---|
| Hamburg | Hamburg |
| Berlin | Berlin |
| Berlin | Leipzig |
| Leipzig | |

- `UNION ALL` – appends **everything**, keeps duplicates, fast.
- `UNION` – appends and **removes duplicates**, needs extra work.

**At Sunny Bay** · `silver/sales_order.sql` stacks **POS receipts** and **web orders** into one `sales_order` table. Both sources are first mapped to the same columns and codes, then combined with `UNION ALL`.

> **💬 Key message:** Use `UNION ALL` by default. Two different orders are never duplicates – removing "duplicates" would delete real sales.

---

## ⚖️ MINUS and INTERSECT – compare lists

```sql
SELECT sku FROM VALUES ('SB-0101'), ('SB-0102'), ('SB-0103') AS master(sku)
MINUS
SELECT sku FROM VALUES ('SB-0102'), ('SB-0103'), ('SB-9999') AS sold(sku);
```

| Operator | Question | Result |
|---|---|---|
| `MINUS` (= `EXCEPT`) | in the first list, but **not** in the second? | SB-0101 |
| `INTERSECT` | in **both** lists? | SB-0102, SB-0103 |

**Typical questions:**
- Which products were **never sold**? *(product master `MINUS` sold products)*
- Which customers bought **in a café and online**? *(café customers `INTERSECT` online customers)*
- Is my new table **identical** to the old one? *(`MINUS` in both directions → 0 rows)*

> **✨ Tip:** "MINUS in both directions" is the standard way to **test** a refactored query: if both directions return **no rows**, the old and the new table are identical.

---

## 🧱 WITH … AS – name your building blocks

**Why?** Nested subqueries are read from the inside out. **CTEs** (common table expressions) are read **top to bottom**, like a recipe.

```sql
WITH prices AS (
    SELECT * FROM VALUES ('Latte', 4.20), ('Espresso', 2.60), ('Flat White', 4.30) AS t(drink, price_eur)
),
expensive AS (
    SELECT * FROM prices WHERE price_eur > 4
)
SELECT COUNT(*) AS expensive_drinks FROM expensive;
```

| expensive_drinks |
|---|
| 2 |

- Every CTE has a **name** that explains what it contains.
- Later CTEs can use earlier ones.
- You can test each block on its own: `SELECT * FROM prices`.

**At Sunny Bay** · `gold/dim_customer.sql` is built from the CTEs `current_card`, `sales`, `store_orders` and `favourite_store` – one business question per block.

---

## 🪟 Window functions – GROUP BY without losing rows

| `GROUP BY` | Window function |
|---|---|
| **collapses** each group to **one** row | keeps **every** row |
| "revenue per store" | "revenue of this month **and** the store total next to it" |

**Anatomy:**

```text
FUNCTION(...) OVER (PARTITION BY <group>  ORDER BY <sequence>)
                    \__ which rows belong  \__ in which order
                        together                within the group
```

| Function | Returns |
|---|---|
| `ROW_NUMBER()` | 1, 2, 3, … per partition |
| `RANK()` | rank with gaps for ties (1, 1, 3) |
| `SUM(x) OVER (…)` | group total – or a **running total** with `ORDER BY` |
| `LAG(x)` / `LEAD(x)` | value of the **previous** / **next** row |

---

## 🪟 Window functions – the example

```sql
SELECT
    store, month, revenue,
    ROW_NUMBER() OVER (PARTITION BY store ORDER BY month)  AS month_no,
    SUM(revenue) OVER (PARTITION BY store)                 AS store_total,
    SUM(revenue) OVER (PARTITION BY store ORDER BY month)  AS running_total,
    LAG(revenue) OVER (PARTITION BY store ORDER BY month)  AS previous_month
FROM VALUES
    ('Hamburg', 1, 100), ('Hamburg', 2, 120), ('Hamburg', 3, 90),
    ('Berlin',  1, 80),  ('Berlin',  2, 95)
    AS t(store, month, revenue)
ORDER BY store, month;
```

| store | month | revenue | month_no | store_total | running_total | previous_month |
|---|---|---|---|---|---|---|
| Berlin | 1 | 80 | 1 | 175 | 80 | `NULL` |
| Berlin | 2 | 95 | 2 | 175 | 175 | 80 |
| Hamburg | 1 | 100 | 1 | 310 | 100 | `NULL` |
| Hamburg | 2 | 120 | 2 | 310 | 220 | 100 |
| Hamburg | 3 | 90 | 3 | 310 | 310 | 120 |

**At Sunny Bay** · `silver/product_price.sql` – the ERP only stores when a price **starts**. `LEAD(valid_from) OVER (PARTITION BY product_id ORDER BY valid_from)` finds the start of the **next** price → the end of the current one.

---

## ➕ ROLLUP – subtotals and a grand total

**Why?** Reports need **subtotals**: per year, per store, overall – without writing three queries.

```sql
SELECT year, month, SUM(revenue) AS revenue, GROUPING(month) AS is_year_total
FROM VALUES (2025, 11, 100), (2025, 12, 150), (2026, 1, 120) AS t(year, month, revenue)
GROUP BY ROLLUP (year, month)
ORDER BY year NULLS LAST, month NULLS LAST;
```

| year | month | revenue | is_year_total |
|---|---|---|---|
| 2025 | 11 | 100 | 0 |
| 2025 | 12 | 150 | 0 |
| 2025 | `NULL` | **250** | 1 |
| 2026 | 1 | 120 | 0 |
| 2026 | `NULL` | **120** | 1 |
| `NULL` | `NULL` | **370** | 1 |

- `ROLLUP (year, month)` → *year + month*, *year*, *grand total* – follows a **hierarchy**.
- `CUBE (store, category)` → **every** combination: per store, per category, per both, total.
- `GROUPING(col) = 1` tells you: "this `NULL` means **all values**", not a missing value.

> **💬 Key message:** In the dashboard, the metric view and AI/BI do this for you. In SQL, `ROLLUP` / `CUBE` give you the same subtotals in one query.

---

## 🌳 WITH RECURSIVE – walk through a hierarchy

**Why?** Org charts and category trees have an **unknown depth**: a manager has employees, who have employees, who …

```sql
WITH RECURSIVE chart (name, level) AS (
    SELECT name, 1                                   -- ① anchor: the top
    FROM VALUES ('Benno', NULL), ('Katrin', 'Benno'), ('Jonas', 'Benno'), ('Mila', 'Katrin') AS t(name, manager)
    WHERE manager IS NULL
    UNION ALL
    SELECT t.name, c.level + 1                       -- ② recursive step: one level deeper
    FROM VALUES ('Benno', NULL), ('Katrin', 'Benno'), ('Jonas', 'Benno'), ('Mila', 'Katrin') AS t(name, manager)
    INNER JOIN chart AS c ON t.manager = c.name
)
SELECT * FROM chart ORDER BY level, name;
```

| name | level |
|---|---|
| Benno | 1 |
| Jonas | 2 |
| Katrin | 2 |
| Mila | 3 |

① starts at the top · ② joins the next level to the rows found so far · stops when a step finds **no new rows**.

**At Sunny Bay** · `hr_employees` (founder Benno Bohne at the top) and `erp_product_categories` (All Products › Drink › Espresso) are both trees.

> **✨ Tip:** If the depth is **fixed and small** (the product categories always have 3 levels), simple self-joins are enough – that is what `gold/dim_product.sql` does.

---

## 🗺️ Cheat sheet

| I want to … | Use |
|---|---|
| stack rows from two sources | `UNION ALL` |
| stack and remove real duplicates | `UNION` |
| find what is missing in a second list | `MINUS` / `EXCEPT` |
| find what is in both lists | `INTERSECT` |
| prove two tables are identical | `MINUS` in both directions → 0 rows |
| make a long query readable | `WITH name AS (…)` |
| number, rank or compare rows **within a group** | `ROW_NUMBER`, `RANK`, `LAG`, `LEAD` + `OVER (PARTITION BY …)` |
| running totals | `SUM(…) OVER (PARTITION BY … ORDER BY …)` |
| subtotals and grand totals | `GROUP BY ROLLUP` / `CUBE` + `GROUPING()` |
| a hierarchy of unknown depth | `WITH RECURSIVE` |

---

## 🏁 Your turn – hands-on

Open the notebook **SQL 2 – Combining and Analyzing Data** and work through it cell by cell:

- Run every example, change it, run it again.
- Solve the **🏋️ Your turn** exercises in the empty cells.
- Stuck? The notebook **SQL Solutions** has every solution.

**Next:** SQL 3 – the functions that turn messy source data into a clean gold layer.
