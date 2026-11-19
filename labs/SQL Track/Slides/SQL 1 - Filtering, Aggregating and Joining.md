# 🔗 SQL 1 – Filtering, Aggregating and Joining

**SQL Track · Presentation for the hands-on lab**

| Today | Concepts |
|---|---|
| 🎯 Pick the right rows | `WHERE`, `NULL` |
| 📊 Summarize | `GROUP BY`, `HAVING` |
| 🔗 Combine tables | `INNER`, `LEFT`, `FULL OUTER`, `SEMI`, `ANTI`, `CROSS JOIN` |

---

## ☕ Where our data comes from

Sunny Bay Roastery runs five cafés (Hamburg, Berlin, München, Köln, Frankfurt am Main) and an online shop. Its data lives in **several operational systems** – just like in a real company.

```text
Source systems         Bronze               Silver                  Gold
POS, Webshop,   --->   raw tables,   --->   business objects  --->  star schema
CRM, ERP, HR,          1:1 copy of          customer, product,      fact_coffee_sales
reference data         the source           sales_order, ...        dim_customer, dim_product, ...
```

- **Bronze** – raw, messy, exactly as delivered (`pos_transactions`, `crm_customers`, `web_orders`, …)
- **Silver** – cleaned and **integrated**: one `customer`, no matter which system knows them
- **Gold** – shaped for reporting: one fact table, several dimensions

> **💬 Key message:** Almost every step from bronze to gold is one of today's patterns: *filter* the right rows, *aggregate* them, *join* systems together.

---

## 🧭 How SQL reads your query

You **write** `SELECT` first – but the database **evaluates** it in this order:

```text
1  FROM / JOIN    which tables?
2  WHERE          which rows?
3  GROUP BY       which groups?
4  HAVING         which groups survive?
5  SELECT         which columns?
6  ORDER BY       in which order?
7  LIMIT          how many rows?
```

> **💬 Key message:** This order explains most "why doesn't this work?" moments – e.g. why `WHERE` cannot use an aggregate and why a column alias from `SELECT` is unknown in `WHERE`.

---

## 🎯 WHERE – pick the right rows

**Why?** Source systems contain rows that must **not** be counted: voided receipts, cancelled orders, test scans.

```sql
SELECT *
FROM VALUES
    ('Single Espresso', 2.60, 'Drink'),
    ('Franzbrötchen',   3.20, 'Food'),
    ('House Blend 1kg', 32.00, 'Beans'),
    ('Iced Latte',      4.60, 'Drink')
    AS t(product, price_eur, category)
WHERE price_eur < 5
  AND category IN ('Drink', 'Food');
```

| product | price_eur | category |
|---|---|---|
| Single Espresso | 2.60 | Drink |
| Franzbrötchen | 3.20 | Food |
| Iced Latte | 4.60 | Drink |

**Toolbox:** `=` `<>` `<` `>` · `BETWEEN` · `IN (…)` · `LIKE 'Sunny%'` · `IS NULL` · `AND` / `OR` / `NOT`

**At Sunny Bay** · `silver/sales_order.sql` keeps only completed receipts: `WHERE status_code = 'C'` (`V` = voided) – the first cleaning step of almost every silver table.

---

## 🕳️ NULL – the missing value

`NULL` means **"unknown"**. Comparing anything with unknown is … unknown – and `WHERE` only keeps rows where the condition is **true**.

| Expression | Result |
|---|---|
| `NULL = NULL` | `NULL` (not true!) |
| `NULL <> 'G'` | `NULL` |
| `NULL IS NULL` | `true` |
| `COUNT(*)` | counts **all** rows |
| `COUNT(loyalty_card_no)` | counts only rows **with** a card |

```sql
-- ❌ returns no rows at all
SELECT * FROM bronze.pos_transactions WHERE loyalty_card_no = NULL;

-- ✅ receipts of guests without a loyalty card
SELECT * FROM bronze.pos_transactions WHERE loyalty_card_no IS NULL;
```

> **⚠️ Watch out:** `WHERE tier <> 'Gold'` also drops all rows where `tier` is `NULL`. If you want them, say so: `WHERE tier <> 'Gold' OR tier IS NULL`.

---

## 📊 GROUP BY – one row per group

**Why?** Reports rarely show single receipts – they show **revenue per store**, **orders per month**, **units per product**.

```sql
SELECT
    category,
    COUNT(*)       AS products,
    AVG(price_eur) AS avg_price_eur,
    MAX(price_eur) AS max_price_eur
FROM VALUES
    ('Single Espresso', 2.60, 'Drink'),
    ('Franzbrötchen',   3.20, 'Food'),
    ('House Blend 1kg', 32.00, 'Beans'),
    ('Iced Latte',      4.60, 'Drink')
    AS t(product, price_eur, category)
GROUP BY category;
```

| category | products | avg_price_eur | max_price_eur |
|---|---|---|---|
| Drink | 2 | 3.60 | 4.60 |
| Food | 1 | 3.20 | 3.20 |
| Beans | 1 | 32.00 | 32.00 |

**Rule:** every column in `SELECT` is either **in the `GROUP BY`** or **inside an aggregate** (`COUNT`, `SUM`, `AVG`, `MIN`, `MAX`). Shortcut on Databricks: `GROUP BY ALL`.

**At Sunny Bay** · `gold/dim_customer.sql` counts the orders and the first and last purchase date per customer with `GROUP BY f.customer_key`.

---

## 🧹 WHERE vs. HAVING

| | `WHERE` | `HAVING` |
|---|---|---|
| filters … | **rows** | **groups** |
| runs … | **before** `GROUP BY` | **after** `GROUP BY` |
| can use aggregates? | ❌ | ✅ |

```sql
SELECT p.product_name, SUM(l.quantity) AS units_sold
FROM silver.sales_order_line AS l
INNER JOIN silver.product AS p ON l.product_id = p.product_id
WHERE l.quantity > 0                 -- rows: ignore the faulty -1 lines
GROUP BY p.product_name
HAVING SUM(l.quantity) < 5000        -- groups: only the slow sellers
ORDER BY units_sold;
```

> **💬 Key message:** Filter as early as possible with `WHERE` – it is cheaper and clearer. Use `HAVING` only for conditions on aggregates.

---

## 🔗 Why joins?

Every system knows **one part** of the truth:

| System | knows … | but not … |
|---|---|---|
| POS | the receipt and the loyalty card number | the customer's name |
| CRM | the customer and their cards | what they bought |
| ERP | product names and prices | how often they were sold |

A **join** puts these parts together using a common **key** (`customer_id`, `product_id`, `card_no`, …). The join **type** decides what happens to rows **without** a partner.

---

## 🧪 Two tiny tables for the first join examples

| `demo_customer` |  |  | `demo_card` |  |  |
|---|---|---|---|---|---|
| **customer_id** | **name** | | **card_no** | **customer_id** | **tier** |
| 1 | Anna | | C-1 | 1 | Gold |
| 2 | Ben | | C-3 | 3 | Silver |
| 3 | Clara | | C-9 | 9 | Bronze |

- **Anna** and **Clara** have a card → they match.
- **Ben** has **no card**.
- Card **C-9** belongs to customer 9, who is **not** in the customer table (an orphan).

---

## 🟣 INNER JOIN – only matching rows

```sql
SELECT c.name, k.card_no, k.tier
FROM sql_sandbox.demo_customer AS c
INNER JOIN sql_sandbox.demo_card AS k
    ON c.customer_id = k.customer_id;
```

| name | card_no | tier |
|---|---|---|
| Anna | C-1 | Gold |
| Clara | C-3 | Silver |

Ben **and** card C-9 are gone – **silently**.

**At Sunny Bay** · `silver/sales_order_line.sql` joins the order lines to `product`: register test scans with the unknown SKU `SB-9999` disappear on purpose.

> **⚠️ Watch out:** An inner join is a filter in disguise. Always ask: *"Which rows do I lose – and is that intended?"* Compare `COUNT(*)` before and after the join.

---

## 🔵 LEFT JOIN – keep everything from the left

```sql
SELECT c.name, k.card_no, k.tier
FROM sql_sandbox.demo_customer AS c
LEFT JOIN sql_sandbox.demo_card AS k
    ON c.customer_id = k.customer_id;
```

| name | card_no | tier |
|---|---|---|
| Anna | C-1 | Gold |
| Ben | `NULL` | `NULL` |
| Clara | C-3 | Silver |

**At Sunny Bay** · `gold/fact_coffee_sales.sql` – order lines `LEFT JOIN` loyalty cards: a guest has no card, but the sale must stay in the fact table. Same in `gold/dim_date.sql`: most days are no public holiday.

> **⚠️ Watch out:** `WHERE k.tier = 'Gold'` after a LEFT JOIN removes Ben again – the LEFT JOIN quietly turns into an INNER JOIN. Conditions on the right table belong into the `ON` clause.

---

## 🟢 FULL OUTER JOIN – compare two systems

```sql
SELECT c.customer_id, c.name, k.card_no, k.customer_id AS card_customer_id
FROM sql_sandbox.demo_customer AS c
FULL OUTER JOIN sql_sandbox.demo_card AS k
    ON c.customer_id = k.customer_id;
```

| customer_id | name | card_no | card_customer_id |
|---|---|---|---|
| 1 | Anna | C-1 | 1 |
| 2 | Ben | `NULL` | `NULL` |
| 3 | Clara | C-3 | 3 |
| `NULL` | `NULL` | C-9 | 9 |

Everything from **both** sides: matches, left-only **and** right-only rows.

**At Sunny Bay** · `silver/customer.sql` combines **CRM** customers and **web shop** accounts on the e-mail address → café-only, online-only and omnichannel customers in one table.

> **⚠️ Watch out:** Keys must be **normalized** before they match: `LOWER(TRIM(crm.email)) = web.email`. `' Anna.Schmidt@Example.com'` and `'anna.schmidt@example.com'` are different strings.

---

## 🧾 Order lines and orders – the example for SEMI and ANTI JOIN

The café register writes **every** receipt – also the ones that were **voided** afterwards. `silver.sales_order` only contains **completed** orders. Which order lines may go to silver?

| `bronze.pos_transaction_lines` |  |  |  | `silver.sales_order` |  |
|---|---|---|---|---|---|
| **transaction_id** | **line_no** | **sku** | | **order_id** | **sales_channel** |
| HH01-20261102-0001 | 1 | SB-0102 | | HH01-20261102-0001 | In-store |
| HH01-20261102-0001 | 2 | SB-0201 | | HH01-20261102-0002 | In-store |
| HH01-20261102-0002 | 1 | SB-0101 | | | |
| HH01-20261102-0003 | 1 | SB-0301 | | *(receipt 0003 was voided)* | |

- Receipt **0001** has **two** lines, receipt **0002** one line – both receipts are completed.
- Receipt **0003** was voided – it has a line, but **no** completed order.

---

## 🟡 LEFT SEMI JOIN – "has at least one match"

**Question:** *Which order lines belong to a **completed** order?* We only want the **line** columns – the order is just a check.

```sql
SELECT l.*
FROM bronze.pos_transaction_lines AS l
LEFT SEMI JOIN silver.sales_order AS o
    ON l.transaction_id = o.order_id;
```

| transaction_id | line_no | sku |
|---|---|---|
| HH01-20261102-0001 | 1 | SB-0102 |
| HH01-20261102-0001 | 2 | SB-0201 |
| HH01-20261102-0002 | 1 | SB-0101 |

*(result for the sample rows – `quantity`, `unit_price`, … are kept as well)*

**Why not an INNER JOIN?** Here it returns the same lines, but it **adds** all order columns – and if the right table ever had two rows for one order, every line would be **duplicated**. A semi join only checks **whether** a partner exists.

Same result in standard SQL: `WHERE EXISTS (SELECT 1 FROM silver.sales_order AS o WHERE o.order_id = l.transaction_id)`.

**At Sunny Bay** · `silver/sales_order_line.sql` does exactly this: `LEFT SEMI JOIN silver.sales_order` keeps only the lines of completed orders.

---

## 🔴 LEFT ANTI JOIN – "has no match"

**Question:** *Which order lines do we **drop** on the way to silver?*

```sql
SELECT l.*
FROM bronze.pos_transaction_lines AS l
LEFT ANTI JOIN silver.sales_order AS o
    ON l.transaction_id = o.order_id;
```

| transaction_id | line_no | sku |
|---|---|---|
| HH01-20261102-0003 | 1 | SB-0301 |

*(result for the sample rows – on the real data: the lines of all voided receipts)*

Same result: `WHERE NOT EXISTS (…)` or `LEFT JOIN … WHERE o.order_id IS NULL`.

**Typical questions:** which rows does a pipeline step **drop** – and is that intended? · loyalty members who **never bought** anything · products that were **never sold** → perfect for data-quality checks.

> **✨ Tip:** SEMI + ANTI always add up: `lines in SEMI JOIN` + `lines in ANTI JOIN` = `all lines`. A quick way to check that no line got lost.

---

## ✖️ CROSS JOIN – every combination

```sql
SELECT d.drink, s.size
FROM VALUES ('Latte'), ('Cappuccino') AS d(drink)
CROSS JOIN VALUES ('Small'), ('Medium'), ('Large') AS s(size);
```

| drink | size |
|---|---|
| Latte | Small |
| Latte | Medium |
| Latte | Large |
| Cappuccino | Small |
| Cappuccino | Medium |
| Cappuccino | Large |

2 × 3 = **6 rows** – no `ON` clause.

**At Sunny Bay** · *store × day* grid: CROSS JOIN all cafés with all days, then LEFT JOIN the sales → the days on which a café sold **nothing** become visible (German public holidays – the cafés are closed).

> **⚠️ Watch out:** 1,000 × 1,000 rows = 1,000,000 rows. Cross join only small tables.

---

## 🗺️ Join cheat sheet

| Join | Keeps | Columns from | Result in our examples | Typical question |
|---|---|---|---|---|
| `INNER` | matches only | both | Anna, Clara | "sales with a known product" |
| `LEFT` | all left + matches | both | Anna, Ben, Clara | "all customers, with card if any" |
| `FULL OUTER` | everything | both | Anna, Ben, Clara, C-9 | "compare two systems" |
| `LEFT SEMI` | left rows **with** a match | left only | lines of receipts 0001 and 0002 | "which lines belong to a completed order?" |
| `LEFT ANTI` | left rows **without** a match | left only | the line of the voided receipt 0003 | "which lines do we drop?" |
| `CROSS` | every combination | both | 3 × 3 = 9 rows | "every store × every day" |

**Style rule in this workshop:** always write the join type (`INNER JOIN`, never a plain `JOIN`) and introduce every alias with `AS`.

---

## 🧯 Three classic join pitfalls

1. **Silently lost rows** – an `INNER JOIN` drops rows without a partner. ➜ Count rows before and after; use `LEFT JOIN` when in doubt.

2. **Messy keys** – different case, blanks, types (`'0101'` vs. `101`). ➜ Normalize keys (`LOWER`, `TRIM`, `CAST`) before joining.

3. **Fan-out** – the right table has **several** rows per key (e.g. the CRM keeps **history versions** of a customer) → every left row is **duplicated** and revenue is counted twice. ➜ Make the right side unique first (→ `QUALIFY` in SQL 3).

---

## 🏁 Your turn – hands-on

Open the notebook **SQL 1 – Filtering, Aggregating and Joining** and work through it cell by cell:

- Run every example, change it, run it again – **experimenting is encouraged**.
- Solve the **🏋️ Your turn** exercises in the empty cells.
- Stuck? The notebook **SQL Solutions** has every solution.

**Next:** SQL 2 – combining result sets and analyzing rows in relation to other rows.
