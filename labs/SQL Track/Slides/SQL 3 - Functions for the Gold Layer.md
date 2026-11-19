# 🧰 SQL 3 – Functions for the Gold Layer

**SQL Track · Presentation for the hands-on lab**

| Problem in the source data | Today's tool |
|---|---|
| Nested arrays (web orders) | `EXPLODE` |
| Several versions of the same record | `QUALIFY` |
| Missing values and unknown keys | `COALESCE` |
| Rules and categories | `CASE`, `IFF` |
| Cryptic one-letter codes | `DECODE` |
| "Latest" and "favourite" values | `MAX_BY`, `MIN_BY` |
| Several KPIs in one pass | `FILTER (WHERE …)` |
| Validity ranges and time spans | `DATEADD`, `DATEDIFF` |

---

## 🤔 Why functions?

Source systems deliver data the way **they** need it – not the way a **report** needs it:

| The web shop sends … | The report needs … |
|---|---|
| one JSON document per order, products in an array | one row per product |
| `payment.method = 'credit_card'` | `Credit Card` |
| the CRM sends every **change** of a customer | one row per customer |
| `discount = NULL` | `0.00 €` |
| no customer for guest orders | a "Guest" bucket |

> **💬 Key message:** Every function today fixes one of these gaps – and every one of them is used in the Sunny Bay pipeline in `bundle/src/transformations/`.

---

## 💥 EXPLODE – one row per array element

```sql
SELECT order_id, EXPLODE(items) AS item
FROM VALUES
    ('A-1', ARRAY('Latte', 'Croissant')),
    ('A-2', ARRAY('Espresso'))
    AS t(order_id, items);
```

| order_id | items | | order_id | item |
|---|---|---|---|---|
| A-1 | [Latte, Croissant] | ➜ | A-1 | Latte |
| A-2 | [Espresso] | | A-1 | Croissant |
| | | | A-2 | Espresso |

- `POSEXPLODE` additionally returns the **position** (0, 1, …) – handy as a line number.
- `EXPLODE(SEQUENCE(start, end, INTERVAL 1 DAY))` generates **one row per day**.

**At Sunny Bay** · `silver/sales_order_line.sql` turns the `items` array of every web order into order lines with `POSEXPLODE` · `gold/dim_date.sql` builds the complete calendar with `EXPLODE(SEQUENCE(…))`.

> **⚠️ Watch out:** Rows with an empty or `NULL` array **disappear**. Use `EXPLODE_OUTER` if you need to keep them.

---

## 🎯 QUALIFY – keep the right row per group

**Problem:** The CRM delivers **every version** of a customer. Joining it would duplicate sales (the fan-out from SQL 1).

```sql
SELECT *
FROM VALUES
    (1, 'Anna', 'Hamburg',  DATE'2024-01-10'),
    (1, 'Anna', 'Lüneburg', DATE'2025-06-01'),
    (2, 'Ben',  'Berlin',   DATE'2024-03-05')
    AS t(customer_id, name, city, updated_at)
QUALIFY ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY updated_at DESC) = 1;
```

| customer_id | name | city | updated_at |
|---|---|---|---|
| 1 | Anna | Lüneburg | 2025-06-01 |
| 2 | Ben | Berlin | 2024-03-05 |

**The pattern:** `QUALIFY ROW_NUMBER() OVER (PARTITION BY <key> ORDER BY <newest first>) = 1`

`QUALIFY` filters on a **window function** – like `HAVING` filters on an aggregate. Without it you need a subquery.

**At Sunny Bay** · `silver/customer.sql` keeps the latest CRM version · `gold/dim_customer.sql` picks each customer's current loyalty card.

---

## 🧭 The full evaluation order

```text
1  FROM / JOIN
2  WHERE           filter rows
3  GROUP BY        build groups
4  HAVING          filter groups
5  window functions  ROW_NUMBER, LAG, SUM(...) OVER (...)
6  QUALIFY         filter on window functions
7  SELECT / DISTINCT
8  ORDER BY
9  LIMIT
```

> **💬 Key message:** `WHERE` filters rows, `HAVING` filters groups, `QUALIFY` filters the result of window functions – three filters for three stages.

---

## 🩹 COALESCE – the first value that is not NULL

```sql
SELECT
    COALESCE(NULL, NULL, 'third')   AS first_non_null,
    COALESCE(NULL, 0)               AS default_zero,
    COALESCE('Hamburg', 'Unknown')  AS keeps_value;
```

| first_non_null | default_zero | keeps_value |
|---|---|---|
| third | 0 | Hamburg |

**Two jobs:**
1. **Default values** – `COALESCE(discount_eur, 0)`
2. **First source that knows it** – `COALESCE(crm.email, web.email)`

**At Sunny Bay** · `gold/fact_coffee_sales.sql`:

```sql
COALESCE(lc.customer_id, wc.customer_id, -1) AS customer_key
--        via loyalty card  via web account  "Guest"
```

> **✨ Tip:** The **unknown member** (`-1` = Guest) is a gold-layer best practice: every fact row finds its dimension row, and reports show a "Guest" bucket instead of silently losing sales in an inner join.

---

## 🔀 CASE and IFF – derive values with rules

```sql
SELECT
    price_eur,
    CASE
        WHEN price_eur < 3  THEN 'cheap'
        WHEN price_eur < 10 THEN 'regular'
        ELSE 'premium'
    END                                  AS price_band,
    IFF(price_eur >= 10, 'yes', 'no')    AS is_premium
FROM VALUES (2.60), (4.20), (32.00) AS t(price_eur);
```

| price_eur | price_band | is_premium |
|---|---|---|
| 2.60 | cheap | no |
| 4.20 | regular | no |
| 32.00 | premium | yes |

- `CASE` checks the conditions **top to bottom** – the **first** true one wins.
- No `ELSE` → `NULL`.
- `IFF(condition, then, else)` is the short form for exactly **two** outcomes.

**At Sunny Bay** · `gold/dim_customer.sql` derives the segments: *Prospect* (no orders), *Loyalist* (≥ 100), *Regular* (≥ 10), otherwise *Occasional*.

---

## 🔤 DECODE – translate codes

```sql
SELECT
    tier_code,
    DECODE(tier_code, 'B', 'Bronze', 'S', 'Silver', 'G', 'Gold', 'Unknown') AS loyalty_tier
FROM VALUES ('G'), ('B'), ('X') AS t(tier_code);
```

| tier_code | loyalty_tier |
|---|---|
| G | Gold |
| B | Bronze |
| X | Unknown |

`DECODE(expr, code1, value1, code2, value2, …, default)` = a compact `CASE expr WHEN … THEN … END` for **lookups**.

**At Sunny Bay** · payment codes (`B` → Cash, `G` → Girocard) in `silver/sales_order.sql` · loyalty tiers in `silver/loyalty_card.sql` · VAT classes (`S` → 19 %, `R` → 7 %) in `silver/product.sql`.

> **✨ Tip:** Always give a **default** (`'Unknown'`). A new code in the source system then shows up visibly instead of as `NULL`.

---

## 🏆 MAX_BY and MIN_BY – the value at the maximum

**Question:** *What is the **current** price?* Not the highest price – the price with the **latest** `valid_from`.

```sql
SELECT
    product,
    MAX(valid_from)            AS latest_change,
    MAX_BY(price, valid_from)  AS current_price,
    MIN_BY(price, valid_from)  AS first_price
FROM VALUES
    ('Latte', DATE'2023-01-01', 3.90),
    ('Latte', DATE'2025-08-01', 4.20),
    ('Espresso', DATE'2023-01-01', 2.40)
    AS t(product, valid_from, price)
GROUP BY product;
```

| product | latest_change | current_price | first_price |
|---|---|---|---|
| Latte | 2025-08-01 | 4.20 | 3.90 |
| Espresso | 2023-01-01 | 2.40 | 2.40 |

`MAX_BY(value, ordering)` = *"give me `value` from the row where `ordering` is highest"* – in one aggregate, without a self-join.

**At Sunny Bay** · current list price in `gold/dim_product.sql` · **favourite store** (the store with the most orders) in `gold/dim_customer.sql`.

---

## 🧺 FILTER (WHERE …) – several KPIs in one pass

```sql
SELECT
    COUNT(*)                                    AS all_orders,
    COUNT(*) FILTER (WHERE channel = 'Online')  AS online_orders,
    SUM(amount) FILTER (WHERE channel = 'Café') AS cafe_revenue
FROM VALUES ('Café', 4.20), ('Online', 32.00), ('Café', 2.60) AS t(channel, amount);
```

| all_orders | online_orders | cafe_revenue |
|---|---|---|
| 3 | 1 | 6.80 |

- Each aggregate gets its **own** condition – the table is read only **once**.
- Same as `SUM(CASE WHEN channel = 'Café' THEN amount END)` – but easier to read.

**At Sunny Bay** · `gold/dim_customer.sql` counts café and online orders per customer with `COUNT(DISTINCT f.order_id) FILTER (WHERE f.sales_channel = …)` → the basis for *In-store* / *Online* / *Omnichannel*.

---

## 📅 DATEADD and DATEDIFF – calculate with dates

```sql
SELECT
    DATEADD(DAY, -1, DATE'2026-01-01')                 AS new_years_eve,
    DATEADD(MONTH, 1, DATE'2026-01-31')                AS one_month_later,
    CAST(DATEADD(YEAR, 1, DATE'2026-03-15') AS DATE)   AS one_year_later,
    DATEDIFF(DAY, DATE'2026-01-01', DATE'2026-12-24')  AS days_until_christmas_eve;
```

| new_years_eve | one_month_later | one_year_later | days_until_christmas_eve |
|---|---|---|---|
| 2025-12-31 00:00:00 | 2026-02-28 00:00:00 | 2027-03-15 | 357 |

- `DATEADD(unit, value, date)` returns a **TIMESTAMP** → `CAST(… AS DATE)` for a date.
- Month ends are handled for you: 31 January + 1 month = **28 February**.

**At Sunny Bay** · `silver/product_price.sql` – a price ends the day **before** the next one starts:

```sql
CAST(DATEADD(DAY, -1, LEAD(valid_from) OVER (PARTITION BY product_id ORDER BY valid_from)) AS DATE) AS valid_to
```

---

## 🗺️ The complete SQL toolbox

| Problem in the source data | Tool | Lab |
|---|---|---|
| Rows that must not be counted | `WHERE` | SQL 1 |
| Data spread over several systems | joins, `UNION ALL` | SQL 1, SQL 2 |
| Comparing systems and tables | `FULL OUTER JOIN`, `MINUS`, `INTERSECT` | SQL 1, SQL 2 |
| Rankings, running totals, previous / next row | window functions | SQL 2 |
| Hierarchies | self-joins, `WITH RECURSIVE` | SQL 2 |
| Nested arrays | `EXPLODE` / `POSEXPLODE` | SQL 3 |
| Several versions of the same record | `QUALIFY ROW_NUMBER() … = 1` | SQL 3 |
| Missing values, unknown keys | `COALESCE`, unknown member `-1` | SQL 3 |
| Cryptic codes, business rules | `DECODE`, `CASE`, `IFF` | SQL 3 |
| "Latest", "favourite" | `MAX_BY` / `MIN_BY` | SQL 3 |
| Several KPIs in one pass | `FILTER (WHERE …)` | SQL 3 |
| Validity ranges, time spans | `LEAD` + `DATEADD`, `DATEDIFF` | SQL 2, SQL 3 |

---

## 🏁 Your turn – hands-on

Open the notebook **SQL 3 – Functions for the Gold Layer** and work through it cell by cell:

- Run every example, change it, run it again.
- Solve the **🏋️ Your turn** exercises in the cells below each step.
- Stuck? The notebook **SQL Solutions** has every solution.
