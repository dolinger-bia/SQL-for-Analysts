# Chapter 5: Joining Tables — Reading a Relational Model

> **SQL for Analysts**
> *A practical introduction to querying, transforming, and understanding relational data*
>
> © Patrick Dolinger, NSCC Institute of Technology
> Licensed under [Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/)
> You are free to share and adapt this material for any purpose, provided appropriate credit is given.

---

## Chapter Overview

Every chapter so far has queried one table at a time. That is enough for the dimension tables — `dim.Product` and `dim.Customer` are human-readable on their own. But `fact.Sales` is not. It contains keys and numbers, and it only becomes meaningful when connected to the dimension tables that provide context.

The `JOIN` is the operation that connects tables. It is the most important concept in relational databases — the mechanism that makes the relational model work. Without `JOIN`, every table is an island. With it, the full picture emerges.

This chapter starts at the beginning — what a join is and why it exists — and builds through the three join types you will use most often: `INNER JOIN`, `LEFT JOIN`, and self-joins. It covers the star schema join pattern in depth, because that is where most analytical SQL queries live.

By the end of this chapter you will be able to:

- Explain what a join is and why it is necessary in a relational database
- Write `INNER JOIN` queries that connect two or more tables
- Use table aliases to write readable multi-table queries
- Write `LEFT JOIN` queries to include rows that have no match
- Describe the difference between `INNER JOIN` and `LEFT JOIN` and choose the right one
- Join a fact table to all of its dimension tables in a star schema
- Recognise and avoid common join mistakes: Cartesian products, fan-out, and ambiguous column names
- Write a self-join to compare rows within a single table

---

## Table of Contents

1. [Why We Need JOINs](#1-why-we-need-joins)
2. [The JOIN Concept: Matching Rows](#2-the-join-concept-matching-rows)
3. [Table Aliases](#3-table-aliases)
4. [INNER JOIN: Matching Rows Only](#4-inner-join-matching-rows-only)
5. [Joining More Than Two Tables](#5-joining-more-than-two-tables)
6. [The Star Schema Join Pattern](#6-the-star-schema-join-pattern)
7. [LEFT JOIN: Keep All Rows from the Left](#7-left-join-keep-all-rows-from-the-left)
8. [Choosing Between INNER JOIN and LEFT JOIN](#8-choosing-between-inner-join-and-left-join)
9. [Common Join Mistakes](#9-common-join-mistakes)
10. [The Self-Join](#10-the-self-join)
11. [Cross-Database Joins](#11-cross-database-joins)
12. [Chapter Summary](#12-chapter-summary)
13. [Review Questions](#13-review-questions)
14. [🔍 Deeper Dive](#-deeper-dive)

---

## 1. Why We Need JOINs

Consider what `fact.Sales` looks like without any joins:

```sql
USE CabotTrailOutdoorsSales;

SELECT TOP 3 * FROM fact.Sales;
```

You will see rows full of numbers and integer keys:

```
SalesKey | OrderDateKey | CustomerKey | ProductKey | EmployeeKey | LineTotal | ...
1        | 20220103     | 15          | 76         | 7           | 499.98    | ...
2        | 20220103     | 8           | 42         | 3           | 84.99     | ...
3        | 20220104     | 22          | 91         | 12          | 1199.97   | ...
```

This tells you almost nothing useful. `CustomerKey = 15` — which customer is that? `ProductKey = 76` — which product? The fact table is intentionally stripped of descriptive information. Those descriptions live in the dimension tables, linked by the keys.

### 1.1 The Problem the Join Solves

The relational model stores each piece of information exactly once. `CustomerName` is stored in `dim.Customer`, not on every row of `fact.Sales`. This is efficient and correct — but it means you need to *connect* the two tables to answer "which customer placed this order?"

The `JOIN` is that connection. It matches rows from one table to rows in another, based on a shared key value. When `fact.Sales.CustomerKey = dim.Customer.CustomerKey`, SQL Server finds the matching customer row and makes its columns available to the query.

### 1.2 The Result

```sql
-- With a JOIN, the fact table becomes readable
SELECT  TOP 3
        cust.CustomerName,
        fs.LineTotal,
        fs.OrderedQuantity
FROM    fact.Sales fs
INNER JOIN dim.Customer cust ON cust.CustomerKey = fs.CustomerKey;
```

```
CustomerName              | LineTotal | OrderedQuantity
Fundy Bay Outfitters      | 499.98    | 2
Trail Hut Ltd             | 84.99     | 1
Cape Breton Outdoor       | 1199.97   | 3
```

Now the data tells a story. Fundy Bay Outfitters bought $499.98 worth of something. We do not yet know what — that requires a second join to `dim.Product`. This is the nature of multi-table queries: each join adds another layer of context.

---

## 2. The JOIN Concept: Matching Rows

### 2.1 What a JOIN Does

A JOIN combines columns from two tables into a single result set. For each row in the first table, it finds matching rows in the second table based on a join condition. The combined columns of both rows form one row in the output.

The join condition is specified in the `ON` clause:

```sql
FROM    fact.Sales fs
INNER JOIN dim.Customer cust
    ON cust.CustomerKey = fs.CustomerKey
```

This says: "For each row in `fact.Sales`, find the row in `dim.Customer` where `CustomerKey` values match. Combine both rows into the output."

### 2.2 Keys Are the Bridge

The join condition almost always connects a **foreign key** (in the child table) to a **primary key** (in the parent table):

- `fact.Sales.CustomerKey` is a foreign key — it references `dim.Customer.CustomerKey`
- `dim.Customer.CustomerKey` is the primary key — it uniquely identifies each customer
- The join finds the one `dim.Customer` row whose `CustomerKey` matches the `CustomerKey` on each `fact.Sales` row

This is why the relational model works: foreign keys create the bridges that JOINs use. Every relationship you identified in the ERD diagrams of DBAS 2010 is a potential join condition.

### 2.3 Visualising the Match

```
fact.Sales:                       dim.Customer:
SalesKey | CustomerKey | ...      CustomerKey | CustomerName | ...
1        | 15          | ...      ...
2        | 8           | ...      8           | Trail Hut Ltd | ...
3        | 22          | ...      ...
                                  15          | Fundy Bay Outfitters | ...
                                  ...
                                  22          | Cape Breton Outdoor | ...

After INNER JOIN ON fact.Sales.CustomerKey = dim.Customer.CustomerKey:
SalesKey | CustomerKey | CustomerName          | LineTotal | ...
1        | 15          | Fundy Bay Outfitters  | 499.98    | ...
2        | 8           | Trail Hut Ltd         | 84.99     | ...
3        | 22          | Cape Breton Outdoor   | 1199.97   | ...
```

The join finds row 1 in `fact.Sales` (CustomerKey = 15), looks up CustomerKey = 15 in `dim.Customer`, finds 'Fundy Bay Outfitters', and combines the two rows.

---

## 3. Table Aliases

Multi-table queries become unwieldy without aliases. A **table alias** is a shorthand name for a table, defined in the `FROM` or `JOIN` clause and used throughout the query.

### 3.1 Defining and Using Aliases

```sql
-- Without aliases: verbose and hard to read
SELECT  fact.Sales.LineTotal,
        dim.Customer.CustomerName
FROM    fact.Sales
INNER JOIN dim.Customer ON dim.Customer.CustomerKey = fact.Sales.CustomerKey;

-- With aliases: concise and readable
SELECT  fs.LineTotal,
        cust.CustomerName
FROM    fact.Sales fs
INNER JOIN dim.Customer cust ON cust.CustomerKey = fs.CustomerKey;
```

The alias is defined by writing it immediately after the table name in `FROM` or `JOIN`. There is no `AS` keyword required (though `FROM fact.Sales AS fs` also works — `AS` is optional for table aliases).

### 3.2 Alias Conventions

Good aliases are short but meaningful:

| Table | Common alias |
|---|---|
| `fact.Sales` | `fs` |
| `dim.Customer` | `cust` |
| `dim.Product` | `prod` |
| `dim.Calendar` | `cal` |
| `dim.Employee` | `emp` |
| `dim.DeliveryMethod` | `dlv` |

When joining the same table multiple times (covered in section 10), you must use different aliases for each instance: `cal1`, `cal2` or `ord_date`, `inv_date`.

### 3.3 Aliases Are Required for Ambiguous Columns

When two joined tables have columns with the same name, you must use aliases to specify which table's column you want:

```sql
-- Both fact.Sales and dim.Customer have a column called CustomerKey
-- Without alias qualification, SQL Server raises an error:
-- "Ambiguous column name 'CustomerKey'"

SELECT  CustomerKey  -- ERROR: which table?
FROM    fact.Sales
INNER JOIN dim.Customer ON dim.Customer.CustomerKey = fact.Sales.CustomerKey;

-- Correct: qualify with alias
SELECT  fs.CustomerKey,     -- fact.Sales version
        cust.CustomerName   -- dim.Customer version
FROM    fact.Sales fs
INNER JOIN dim.Customer cust ON cust.CustomerKey = fs.CustomerKey;
```

The professional habit is to **always qualify column names with table aliases** in multi-table queries — not just for ambiguous columns. It makes queries self-documenting: a reader can immediately see which table every column comes from.

---

## 4. INNER JOIN: Matching Rows Only

The `INNER JOIN` is the most common join type. It returns only rows where a match is found in both tables. Rows from either table that have no match in the other are excluded.

### 4.1 Syntax

```sql
SELECT  columns
FROM    TableA alias_a
INNER JOIN TableB alias_b ON alias_b.key = alias_a.key;
```

The keyword `INNER` is technically optional — `JOIN` alone defaults to `INNER JOIN`. However, writing `INNER JOIN` explicitly is better practice: it communicates your intent and makes the join type visible when reading the query.

### 4.2 A Complete INNER JOIN Example

```sql
-- Which products were sold, and what category are they in?
USE CabotTrailOutdoorsSales;

SELECT  prod.ProductName,
        prod.CategoryName,
        fs.OrderedQuantity,
        fs.UnitPrice,
        fs.LineTotal
FROM    fact.Sales fs
INNER JOIN dim.Product prod ON prod.ProductKey = fs.ProductKey
ORDER BY prod.CategoryName, fs.LineTotal DESC;
```

This returns 16,359 rows — one per invoice line. Each row combines a `fact.Sales` row with the matching `dim.Product` row, giving both the transaction measures (quantity, price, total) and the product descriptions (name, category).

### 4.3 Adding a WHERE Clause

`WHERE` filters rows *after* the join is applied:

```sql
-- Sales of Sleeping products only
SELECT  prod.ProductName,
        prod.CategoryName,
        fs.OrderedQuantity,
        fs.LineTotal
FROM    fact.Sales fs
INNER JOIN dim.Product prod ON prod.ProductKey = fs.ProductKey
WHERE   prod.CategoryName = 'Sleeping'
ORDER BY fs.LineTotal DESC;
```

The sequence: SQL Server joins the tables first, producing the combined row set, then filters with `WHERE`. You can filter on columns from either table.

### 4.4 What INNER JOIN Excludes

The exclusion behaviour of `INNER JOIN` is important to understand. If `fact.Sales` has a row with `ProductKey = 999` and there is no row in `dim.Product` with `ProductKey = 999`, that `fact.Sales` row is silently excluded from the result.

In a well-designed dimensional model, this should not happen — every FK value in the fact table has a corresponding PK in the dimension (referential integrity). But if data quality is imperfect, `INNER JOIN` can silently drop rows. This is one reason why reconciliation tests (comparing row counts between source and joined result) matter.

---

## 5. Joining More Than Two Tables

Most analytical queries join three, four, or more tables. Each additional `JOIN` adds another connection to the query.

### 5.1 Chaining JOINs

Each `JOIN` clause adds one more table. They are evaluated left to right, with each join extending the result set of all previous joins:

```sql
-- Sales with customer AND product information
SELECT  cust.CustomerName,
        cust.SalesTerritory,
        prod.ProductName,
        prod.CategoryName,
        fs.OrderedQuantity,
        fs.LineTotal
FROM    fact.Sales fs
INNER JOIN dim.Customer cust ON cust.CustomerKey = fs.CustomerKey
INNER JOIN dim.Product  prod ON prod.ProductKey  = fs.ProductKey
ORDER BY cust.SalesTerritory, prod.CategoryName;
```

SQL Server processes this as:
1. Join `fact.Sales` to `dim.Customer` on `CustomerKey`
2. Join the result to `dim.Product` on `ProductKey`
3. Apply `ORDER BY`
4. Return the result

### 5.2 Adding the Calendar Dimension

```sql
-- Sales with customer, product, and date
SELECT  cal.FullDate,
        cal.MonthName,
        cal.CalendarYear,
        cust.CustomerName,
        prod.ProductName,
        prod.CategoryName,
        fs.LineTotal
FROM    fact.Sales fs
INNER JOIN dim.Calendar  cal  ON cal.DateKey   = fs.OrderDateKey
INNER JOIN dim.Customer  cust ON cust.CustomerKey = fs.CustomerKey
INNER JOIN dim.Product   prod ON prod.ProductKey  = fs.ProductKey
WHERE   cal.CalendarYear = 2024
ORDER BY cal.FullDate, cust.CustomerName;
```

Notice that the `WHERE` clause now filters on `cal.CalendarYear` — a column from `dim.Calendar`, not from `fact.Sales`. Once a table is joined, all of its columns are available in `WHERE`, `ORDER BY`, and `SELECT`.

---

## 6. The Star Schema Join Pattern

The CabotTrail Sales data mart is a star schema — a central fact table surrounded by dimension tables. Querying it effectively means understanding the full star join pattern.

### 6.1 The Complete Five-Dimension Join

```sql
-- The full star join: fact.Sales connected to all five dimensions
SELECT
    -- Date dimension
    cal.FullDate                    AS OrderDate,
    cal.MonthName,
    cal.CalendarYear,
    -- Customer dimension
    cust.CustomerName,
    cust.SalesTerritory,
    -- Product dimension
    prod.ProductName,
    prod.CategoryName,
    -- Employee dimension
    emp.FullName                    AS SalesRep,
    emp.Department,
    -- Delivery method dimension
    dlv.DeliveryMethodName,
    -- Fact measures
    fs.OrderedQuantity,
    fs.UnitPrice,
    fs.LineTotal,
    fs.GrossProfit,
    fs.GrossProfitMarginPct
FROM    fact.Sales fs
INNER JOIN dim.Calendar       cal  ON cal.DateKey          = fs.OrderDateKey
INNER JOIN dim.Customer       cust ON cust.CustomerKey     = fs.CustomerKey
INNER JOIN dim.Product        prod ON prod.ProductKey      = fs.ProductKey
INNER JOIN dim.Employee       emp  ON emp.EmployeeKey      = fs.EmployeeKey
INNER JOIN dim.DeliveryMethod dlv  ON dlv.DeliveryMethodKey = fs.DeliveryMethodKey
ORDER BY cal.FullDate, cust.CustomerName;
```

This is the foundational analytical query for the Sales data mart. Once you have this structure, you can answer virtually any question about CabotTrail's sales by adjusting the `SELECT` columns, the `WHERE` filters, and (in Chapter 6) adding `GROUP BY` aggregation.

### 6.2 Reading the Star Join

The visual pattern of the star join query mirrors the star diagram:
- `fact.Sales` is always in the `FROM` clause — the centre of the star
- Each `INNER JOIN` connects one spoke — one dimension — to the centre
- The `ON` condition matches the FK on the fact to the PK on the dimension
- The `SELECT` list picks columns from wherever they are most useful

### 6.3 Selective Joins: You Do Not Have to Join Everything

You only need to join dimensions whose columns you actually use. If a query only needs date and product information, there is no reason to join `dim.Customer`, `dim.Employee`, and `dim.DeliveryMethod`:

```sql
-- Monthly revenue by product category — no need for Customer or Employee joins
SELECT
    cal.MonthName,
    cal.MonthNumber,
    cal.CalendarYear,
    prod.CategoryName,
    COUNT(*)                AS SalesLines,
    SUM(fs.LineTotal)       AS TotalRevenue
FROM    fact.Sales fs
INNER JOIN dim.Calendar cal  ON cal.DateKey  = fs.OrderDateKey
INNER JOIN dim.Product  prod ON prod.ProductKey = fs.ProductKey
WHERE   cal.CalendarYear = 2024
GROUP BY cal.CalendarYear, cal.MonthNumber, cal.MonthName, prod.CategoryName
ORDER BY cal.MonthNumber, prod.CategoryName;
```

> This query uses `GROUP BY` and aggregate functions — covered in Chapter 6. Run it now to see the result, even if the aggregation syntax is new.

Unnecessary joins add processing overhead and can introduce unexpected row multiplication if the join condition is not tight. Join only what you need.

---

## 7. LEFT JOIN: Keep All Rows from the Left

The `INNER JOIN` excludes rows with no match. The `LEFT JOIN` is different: it keeps all rows from the *left* table and adds columns from the *right* table where a match exists. Where there is no match, the right table's columns appear as `NULL`.

### 7.1 Syntax

```sql
SELECT  columns
FROM    TableA alias_a               -- Left table: ALL rows kept
LEFT JOIN TableB alias_b             -- Right table: NULL where no match
    ON alias_b.key = alias_a.key;
```

`LEFT OUTER JOIN` and `LEFT JOIN` are synonyms. `OUTER` is optional and rarely written.

### 7.2 LEFT JOIN in Practice

The most common use of `LEFT JOIN` in analytical work: "show everything in this dimension, even if it has no transactions."

```sql
-- All products, whether or not they have been sold
-- Products with no sales will show NULL for fact columns
SELECT  prod.ProductName,
        prod.CategoryName,
        prod.UnitPrice,
        COUNT(fs.SalesKey)      AS TimesSold,
        SUM(fs.LineTotal)       AS TotalRevenue
FROM    dim.Product prod
LEFT JOIN fact.Sales fs ON fs.ProductKey = prod.ProductKey
GROUP BY prod.ProductName, prod.CategoryName, prod.UnitPrice
ORDER BY TimesSold ASC;
-- Products with zero sales appear at the top (COUNT returns 0 for no matches)
```

Notice the table order is reversed from the previous examples: `dim.Product` is in the `FROM` clause (the left table) and `fact.Sales` is in the `LEFT JOIN` (the right table). This ensures all products are kept, with sales data added where available.

### 7.3 Visualising LEFT JOIN

```
dim.Product (left — all kept):     fact.Sales (right — NULL where no match):
ProductKey | ProductName           ProductKey | SalesKey | LineTotal
76         | Sleeping Bag          76         | 1        | 499.98
83         | Tent                  76         | 2        | 499.98
99         | Widget (never sold)   83         | 3        | 1199.97

After LEFT JOIN:
ProductKey | ProductName   | SalesKey | LineTotal
76         | Sleeping Bag  | 1        | 499.98
76         | Sleeping Bag  | 2        | 499.98
83         | Tent          | 3        | 1199.97
99         | Widget        | NULL     | NULL    ← kept with NULLs
```

The Widget product has no sales — its `SalesKey` and `LineTotal` are `NULL`, but the product row itself is preserved.

### 7.4 Using LEFT JOIN to Find Non-Matches

A common analytical pattern: find rows in one table that have no corresponding rows in another. The technique is `LEFT JOIN` + `WHERE right_table.key IS NULL`:

```sql
-- Which products have never been sold?
SELECT  prod.ProductName,
        prod.CategoryName,
        prod.UnitPrice
FROM    dim.Product prod
LEFT JOIN fact.Sales fs ON fs.ProductKey = prod.ProductKey
WHERE   fs.SalesKey IS NULL     -- NULL SalesKey means no matching sales row
ORDER BY prod.CategoryName, prod.ProductName;
```

The logic: after the `LEFT JOIN`, products with no sales have `NULL` for every `fact.Sales` column. Filtering `WHERE fs.SalesKey IS NULL` isolates exactly those no-match rows.

This pattern — left join plus NULL check — is often more readable than a subquery with `NOT IN` or `NOT EXISTS`, and it performs well.

---

## 8. Choosing Between INNER JOIN and LEFT JOIN

The join type should reflect the analytical question:

| Question type | Join to use | Why |
|---|---|---|
| "What did [X] do?" | `INNER JOIN` | Only interested in X entities that have activity |
| "Show all [X], with [Y] where available" | `LEFT JOIN` | All X entities, regardless of Y activity |
| "Which [X] have no [Y]?" | `LEFT JOIN` + `WHERE Y IS NULL` | Explicitly finding non-matches |
| "Which [X] have at least one [Y]?" | `INNER JOIN` | The inner join naturally excludes X with no Y |

### 8.1 A Concrete Decision

**Question A:** "What revenue did each customer generate in 2024?"

Answer: `INNER JOIN` — you only care about customers who actually bought something in 2024. Customers with no 2024 purchases are irrelevant to this question.

```sql
SELECT  cust.CustomerName,
        SUM(fs.LineTotal)   AS Revenue2024
FROM    fact.Sales fs
INNER JOIN dim.Customer cust ON cust.CustomerKey = fs.CustomerKey
INNER JOIN dim.Calendar  cal ON cal.DateKey       = fs.OrderDateKey
WHERE   cal.CalendarYear = 2024
GROUP BY cust.CustomerName
ORDER BY Revenue2024 DESC;
```

**Question B:** "Show all customers and their 2024 revenue — including customers with no 2024 purchases (show $0 for them)."

Answer: `LEFT JOIN` from `dim.Customer` — you want all customers regardless of purchase activity:

```sql
SELECT  cust.CustomerName,
        ISNULL(SUM(fs.LineTotal), 0)    AS Revenue2024
FROM    dim.Customer cust
LEFT JOIN fact.Sales fs
    ON  fs.CustomerKey  = cust.CustomerKey
    AND fs.OrderDateKey BETWEEN 20240101 AND 20241231   -- Date filter in ON clause
GROUP BY cust.CustomerName
ORDER BY Revenue2024 DESC;
```

> **Important technique:** The date filter is in the `ON` clause rather than the `WHERE` clause. A `WHERE` filter on a left-joined table's column converts the `LEFT JOIN` back into an `INNER JOIN` — because `WHERE` excludes `NULL` rows, and customers with no sales have `NULL` for all `fact.Sales` columns, including the date key.
>
> The rule: when filtering on the right table of a `LEFT JOIN`, put the filter in the `ON` clause, not the `WHERE` clause.

---

## 9. Common Join Mistakes

### 9.1 The Cartesian Product: The Missing ON Clause

A **Cartesian product** (also called a cross join) occurs when a join condition is missing or incorrect, causing every row in one table to match every row in the other:

```sql
-- WRONG: missing ON clause
SELECT cust.CustomerName, prod.ProductName
FROM   dim.Customer cust, dim.Product prod;
-- Returns: 100 customers × 142 products = 14,200 rows
-- Every customer appears with every product — nonsensical
```

With the correct `ON` clause:

```sql
-- This has no natural FK relationship between Customer and Product
-- You cannot meaningfully join them directly — this query has no correct form
```

If you ever see a query return far more rows than expected — millions of rows from two small tables — suspect a missing or incorrect join condition.

### 9.2 Fan-Out: The Wrong Join Column

**Fan-out** occurs when a join multiplies rows unintentionally because the join column is not the unique primary key of the right table. It is more subtle than a Cartesian product:

```sql
-- INCORRECT: joining on a non-unique column
-- Suppose dim.Calendar had two rows for the same year (it does not, but imagine)
-- Any join on CalendarYear instead of DateKey would multiply fact rows
SELECT fs.InvoiceID, cal.FullDate
FROM   fact.Sales fs
INNER JOIN dim.Calendar cal ON cal.YearNumber = YEAR(fs.OrderDate);
-- If cal has multiple rows per year (it has ~365), each fact row
-- matches every day in that year — 365x fan-out
```

```sql
-- CORRECT: join on the actual key
SELECT fs.InvoiceID, cal.FullDate
FROM   fact.Sales fs
INNER JOIN dim.Calendar cal ON cal.DateKey = fs.OrderDateKey;
-- Each fact row matches exactly one calendar row
```

**Detection:** Compare the row count of a joined query to the row count of the primary table. If the join produced more rows than the primary table, fan-out has occurred.

```sql
-- Detect fan-out: should return 16,359 for FactSales
SELECT COUNT(*) FROM fact.Sales;                            -- Source count
SELECT COUNT(*) FROM fact.Sales fs                          -- After join
INNER JOIN dim.Calendar cal ON cal.DateKey = fs.OrderDateKey;
-- If these match, no fan-out
```

### 9.3 Ambiguous Column Names

When joining tables with shared column names, failing to qualify column names produces an error:

```sql
-- Both tables have CustomerKey — this is ambiguous
SELECT CustomerKey, CustomerName, LineTotal
FROM   fact.Sales
INNER JOIN dim.Customer ON dim.Customer.CustomerKey = fact.Sales.CustomerKey;
-- Error: Ambiguous column name 'CustomerKey'

-- Solution: always qualify with alias
SELECT  fs.CustomerKey, cust.CustomerName, fs.LineTotal
FROM    fact.Sales fs
INNER JOIN dim.Customer cust ON cust.CustomerKey = fs.CustomerKey;
```

The professional habit: qualify every column in multi-table queries, even those that are not ambiguous. It costs nothing and prevents errors when tables gain new columns.

### 9.4 Filtering Right-Table Columns in a LEFT JOIN's WHERE Clause

As noted in section 8.1, filtering a right-table column in `WHERE` converts a `LEFT JOIN` to an `INNER JOIN`:

```sql
-- WRONG: this returns only customers who have 2024 sales
-- (The WHERE clause excludes NULL rows from the left join)
SELECT  cust.CustomerName, fs.LineTotal
FROM    dim.Customer cust
LEFT JOIN fact.Sales fs ON fs.CustomerKey = cust.CustomerKey
WHERE   fs.OrderDateKey >= 20240101;   -- Excludes NULL OrderDateKey rows

-- RIGHT: put the date filter in the ON clause
SELECT  cust.CustomerName, fs.LineTotal
FROM    dim.Customer cust
LEFT JOIN fact.Sales fs
    ON  fs.CustomerKey  = cust.CustomerKey
    AND fs.OrderDateKey >= 20240101;   -- Customers with no 2024 sales still appear
```

This is perhaps the most common `LEFT JOIN` mistake. When you write `WHERE rightTable.column = value` after a `LEFT JOIN`, you have effectively written an `INNER JOIN`.

---

## 10. The Self-Join

A **self-join** joins a table to itself. It is used when rows within the same table have a relationship to other rows in the same table — typically hierarchical data (an employee's manager is also an employee) or comparison scenarios.

### 10.1 Syntax

The table is referenced twice, each time with a different alias:

```sql
SELECT  e.FullName          AS Employee,
        mgr.FullName        AS Manager
FROM    Application.Employees e
INNER JOIN Application.Employees mgr ON mgr.EmployeeID = e.ManagerID;
```

Here `e` and `mgr` both refer to the same `Employees` table, but SQL Server treats them as separate instances. `e` provides the employee row; `mgr` provides the manager row (found by looking up the employee's `ManagerID` as another employee's `EmployeeID`).

### 10.2 Self-Join in the CabotTrail Context

The CabotTrail dimension tables do not have hierarchical structures that require self-joins. But the pattern is common in OLTP databases with org charts, bill-of-materials structures, or category parent-child relationships.

For a demonstration, we can compare product prices within the same category — finding products whose price exceeds the average for their category:

```sql
-- Products priced higher than any other product in the same category
-- (An alternative to correlated subqueries — covered in Chapter 7)
SELECT  p1.ProductName,
        p1.CategoryName,
        p1.UnitPrice,
        p2.ProductName      AS CheaperProduct,
        p2.UnitPrice        AS CheaperPrice
FROM    dim.Product p1
INNER JOIN dim.Product p2
    ON  p2.CategoryName = p1.CategoryName   -- Same category
    AND p2.ProductKey   <> p1.ProductKey    -- Different product
    AND p2.UnitPrice    < p1.UnitPrice      -- p2 is cheaper than p1
ORDER BY p1.CategoryName, p1.UnitPrice DESC;
```

This shows every (expensive product, cheaper product) pair within the same category. It is not a typical daily-use query, but it illustrates that the self-join is simply a join with two aliases pointing to the same table.

---

## 11. Cross-Database Joins

SQL Server allows joining tables from different databases on the same server using **three-part naming**: `DatabaseName.SchemaName.TableName`.

### 11.1 Syntax

```sql
-- Join dim.Customer from the Sales datamart to DimCustomer in the DW
SELECT
    sm.CustomerName,
    sm.SalesTerritory,
    dw.CreditLimit
FROM    CabotTrailOutdoorsSales.dim.Customer sm
INNER JOIN CabotTrailOutdoorDW.Dimension.DimCustomer dw
    ON dw.CustomerID = sm.CustomerID
WHERE   sm.ProvinceCode = 'NS'
ORDER BY sm.CustomerName;
```

### 11.2 When Cross-Database Joins Are Appropriate

Cross-database queries are useful for:
- **Reconciliation:** Comparing data in different databases (datamart vs DW)
- **Supplementing a query:** When a column needed for a report lives in a different database
- **ETL verification:** Confirming that data was correctly moved from source to target

Cross-database joins have performance implications — SQL Server must retrieve data across database boundaries. For large tables, it may be more efficient to bring the data into a single database first (using a staging table).

### 11.3 The Sales Datamart vs the Full DW

You will increasingly work with both `CabotTrailOutdoorsSales` (the simplified mart) and `CabotTrailOutdoorDW` (the full warehouse) as the course progresses. The schemas differ:

| `CabotTrailOutdoorsSales` | `CabotTrailOutdoorDW` | Notes |
|---|---|---|
| `dim.Customer` | `Dimension.DimCustomer` | Same data, different schema/naming |
| `dim.Product` | `Dimension.DimProduct` | DW has a snowflake (more joins needed) |
| `fact.Sales` | `Fact.FactSales` | Same data, same structure |
| `dim.Calendar` | `Dimension.DimDate` | Similar content, some column differences |

Three-part naming lets you use both in the same query.

---

## 12. Chapter Summary

- **JOINs** connect rows from two tables based on matching column values. They are the mechanism that makes the relational model useful — combining information stored in separate tables into a coherent result.

- Every join requires an `ON` clause that specifies the matching condition. The condition almost always connects a **foreign key** in one table to the **primary key** in another.

- **Table aliases** are shorthand names assigned in `FROM` and `JOIN` clauses. Always qualify column names with aliases in multi-table queries.

- **INNER JOIN** returns only rows where a match exists in both tables. Rows with no match are excluded from the result.

- **LEFT JOIN** keeps all rows from the left table, adding right-table columns where a match exists and `NULL` where there is none. Used when you want all rows from one table regardless of matches.

- **The star schema join pattern** places the fact table in `FROM` and each dimension table in an `INNER JOIN`. Join only the dimensions whose columns you actually need.

- **Common mistakes:** Cartesian products (missing `ON` clause), fan-out (joining on a non-unique column), ambiguous column names (missing alias qualification), and filtering a `LEFT JOIN`'s right table in `WHERE` instead of `ON`.

- **Self-joins** use two aliases on the same table to compare rows within the table — used for hierarchies and within-table comparisons.

- **Cross-database joins** use three-part naming (`Database.Schema.Table`) to join tables from different databases on the same server.

---

## 13. Review Questions

1. Explain in your own words why `fact.Sales` is not useful without joining dimension tables. What specific information is missing from the fact table alone, and where does it live?

2. Write a query that returns `CustomerName`, `SalesTerritory`, `DeliveryMethodName`, `ProductName`, `CategoryName`, `OrderDate` (from `dim.Calendar.FullDate`), and `LineTotal` for all sales in the Atlantic — Nova Scotia territory in 2023. Join all necessary dimensions and filter appropriately.

3. Write a query that shows all 50 employees and the number of sales transactions each has handled (from `fact.Sales`). Include employees with zero sales — they should appear with a count of 0. Use an appropriate alias for the count column. Sort by transaction count descending.

4. Explain the difference between placing a filter condition in the `ON` clause versus the `WHERE` clause of a `LEFT JOIN`. Write an example showing both versions and explain the different results they produce.

5. A developer writes this query and is surprised when it returns over 1.6 million rows instead of 16,359:
   ```sql
   SELECT fs.InvoiceID, cal.FullDate, fs.LineTotal
   FROM   fact.Sales fs
   INNER JOIN dim.Calendar cal ON cal.CalendarYear = YEAR(fs.OrderDate);
   ```
   Explain the problem, calculate how many rows are expected, and write the correct query.

6. Write a query using a `LEFT JOIN` and a `WHERE IS NULL` filter to find all products in `dim.Product` that have never appeared in `fact.Sales`. Include `ProductName`, `CategoryName`, and `UnitPrice`. Sort by category then product name.

7. Write a query against `CabotTrailOutdoorDW` that joins `Fact.FactSales` to `Dimension.DimDate` (using `OrderDateKey = DateKey`) and `Dimension.DimProduct`. Return `FullDate`, `ProductName`, `CategoryName`, `OrderedQuantity`, and `LineTotal` for the month of March 2024. Use the DW's column naming conventions (`Dimension.DimDate.FullDate`, not `dim.Calendar.FullDate`).

8. Without running it, describe what this query returns and explain whether the join type is appropriate for the question being asked: *"Which delivery methods have never been used for an order over $500?"*
   ```sql
   SELECT  dlv.DeliveryMethodName,
           COUNT(fs.SalesKey)  AS HighValueOrders
   FROM    dim.DeliveryMethod dlv
   LEFT JOIN fact.Sales fs
       ON  fs.DeliveryMethodKey = dlv.DeliveryMethodKey
       AND fs.LineTotal > 500
   GROUP BY dlv.DeliveryMethodName
   ORDER BY HighValueOrders ASC;
   ```

---

## 🔍 Deeper Dive

### Going Further with JOINs

#### The Full JOIN Family

This chapter covered `INNER JOIN` and `LEFT JOIN` — the two you will use in 95% of analytical queries. SQL Server supports several others:

**RIGHT JOIN:** The mirror image of `LEFT JOIN` — keeps all rows from the right table, adds left table columns where a match exists. `RIGHT JOIN` is almost never needed in practice because you can always rewrite it as a `LEFT JOIN` by swapping the table order. Most teams adopt a convention of always using `LEFT JOIN` rather than mixing left and right.

**FULL OUTER JOIN:** Returns all rows from both tables. Rows with no match in the other table get NULLs for the missing side. Useful for comparing two datasets and finding rows that exist in one but not the other:

```sql
-- Find customers who exist in the Sales Mart but not the DW, and vice versa
SELECT
    sm.CustomerName     AS SalesMart_Name,
    dw.CustomerName     AS DW_Name
FROM    CabotTrailOutdoorsSales.dim.Customer sm
FULL OUTER JOIN CabotTrailOutdoorDW.Dimension.DimCustomer dw
    ON dw.CustomerID = sm.CustomerID
WHERE   sm.CustomerID IS NULL   -- In DW but not mart
OR      dw.CustomerID IS NULL;  -- In mart but not DW
```

**CROSS JOIN:** Returns every row from the left table combined with every row from the right table — a deliberate Cartesian product. Occasionally used for generating test data, date grids, or all possible combinations of two sets:

```sql
-- Generate all month/year combinations from the calendar
SELECT DISTINCT cal.YearNumber, m.MonthNumber
FROM   dim.Calendar cal
CROSS JOIN (VALUES (1),(2),(3),(4),(5),(6),(7),(8),(9),(10),(11),(12))
           AS m(MonthNumber)
ORDER BY cal.YearNumber, m.MonthNumber;
```

Microsoft documentation on join types:
[FROM clause plus JOIN, APPLY, PIVOT](https://learn.microsoft.com/en-us/sql/t-sql/queries/from-transact-sql)

#### Join Algorithms: How SQL Server Executes Joins

Understanding how SQL Server *executes* joins helps you write faster queries. The query optimizer chooses one of three join algorithms based on table sizes, indexes, and statistics:

**Nested Loop Join:** For each row in the outer table, scan the inner table for matches. Efficient when the outer table is small and the inner table has an index on the join column. Common for joining a small result set to a large indexed table.

**Hash Join:** Build a hash table from the smaller table, then probe it with rows from the larger table. Efficient for large, unsorted tables with no useful indexes. Memory-intensive.

**Merge Join:** Sort both inputs on the join key, then scan them simultaneously. Extremely efficient when both inputs are already sorted (e.g., from an index). Requires sorted inputs — SQL Server may add a sort operation if they are not.

You can see which algorithm SQL Server chose using the **Execution Plan** (`Ctrl+M` to toggle in SSMS, or `Ctrl+L` for estimated plan). The icons for nested loop, hash match, and merge join tell you the algorithm used for each join in the query.

For analytical queries against the CabotTrail data mart (small dimension tables), nested loop joins are typical and fast. For large fact tables in production environments, the optimizer choice matters significantly.

#### The ANSI SQL Join Syntax vs the Old Comma Syntax

You may encounter older SQL code that uses commas in the `FROM` clause to join tables, with the join condition in the `WHERE` clause:

```sql
-- OLD-STYLE (implicit join — avoid this):
SELECT  fs.LineTotal, cust.CustomerName
FROM    fact.Sales fs, dim.Customer cust
WHERE   cust.CustomerKey = fs.CustomerKey
AND     fs.LineTotal > 500;

-- MODERN (explicit join — always prefer this):
SELECT  fs.LineTotal, cust.CustomerName
FROM    fact.Sales fs
INNER JOIN dim.Customer cust ON cust.CustomerKey = fs.CustomerKey
WHERE   fs.LineTotal > 500;
```

The comma syntax (implicit join) has two problems:
1. **Readability:** Join conditions and filter conditions are mixed in `WHERE` — it is easy to miss one and produce an accidental Cartesian product
2. **Explicit intent:** `INNER JOIN`, `LEFT JOIN`, and `FULL OUTER JOIN` cannot be expressed with the comma syntax; it only supports inner joins

The ANSI SQL join syntax (with explicit `JOIN` keywords) has been standard since SQL-92. All modern SQL code should use it. If you encounter the comma syntax in legacy code, consider rewriting it.

#### Hash JOIN and Performance on Large Tables

For very large fact tables — hundreds of millions of rows — join performance depends critically on proper indexing. The most important indexes for analytical joins:

**On the fact table:** Non-clustered indexes on all FK columns (`CustomerKey`, `ProductKey`, `OrderDateKey`, etc.). Without these, every join to the fact table requires a full table scan.

**On the dimension tables:** The primary key (surrogate key) should have a clustered or non-clustered unique index. SQL Server creates this automatically with a `PRIMARY KEY` constraint.

**Columnstore indexes:** For large fact tables (10M+ rows), a clustered columnstore index transforms analytical query performance — aggregations over many rows are typically 10–100x faster than with traditional row-store indexes.

The CabotTrail fact table at 16,359 rows is too small to benefit from any of this. But understanding these principles prepares you for production environments where join performance matters.

---

### Industry Perspectives

#### JOINs Are the Core Skill

Data analyst job postings consistently list "strong SQL skills" as a requirement. When interviewers assess SQL proficiency, joins are the primary test — specifically:

- Can you write a correct `INNER JOIN`?
- Do you understand the difference between `INNER JOIN` and `LEFT JOIN`?
- Can you explain what happens when a join key has no match?
- Can you debug a query returning unexpected row counts?

The concepts in this chapter — join types, the ON clause, fan-out, the LEFT JOIN filter trap — are the subject of SQL interview questions at virtually every data role. Mastering them is a direct career investment.

#### The Star Schema and BI Tools

Power BI, Tableau, and other BI tools are designed around the star schema join pattern. When you connect Power BI to a star schema data mart, it can automatically detect the FK-to-PK relationships between the fact and dimension tables and propose join conditions. The visual report builder then translates user interactions (drag a field, apply a filter) into the equivalent `JOIN`-and-`WHERE` SQL queries.

Understanding star schema joins in SQL gives you the ability to verify and debug what BI tools are actually doing — an essential skill when a report produces unexpected results and you need to determine whether the issue is in the data or the report logic.

---

### References and Further Reading

1. Ben-Gan, I. (2015). *T-SQL Fundamentals* (3rd ed.). Microsoft Press. — Chapter 3 covers joins comprehensively with clear visual examples of each join type.

2. Itzik Ben-Gan, Lubor Kollar, Dejan Sarka, & Ron Talmage. (2015). *T-SQL Querying*. Microsoft Press. — Chapters 1–2 cover joins and the relational model at depth, including physical join algorithms.

3. Kimball, R., & Ross, M. (2013). *The Data Warehouse Toolkit* (3rd ed.). Wiley. — Chapter 1 explains the star schema join pattern from the designer's perspective, complementing the analyst's view in this chapter.

4. Microsoft. (2024). *FROM clause plus JOIN, APPLY, PIVOT (Transact-SQL)*. [https://learn.microsoft.com/en-us/sql/t-sql/queries/from-transact-sql](https://learn.microsoft.com/en-us/sql/t-sql/queries/from-transact-sql)

5. Microsoft. (2024). *Joins*. [https://learn.microsoft.com/en-us/sql/relational-databases/performance/joins](https://learn.microsoft.com/en-us/sql/relational-databases/performance/joins)

6. Microsoft. (2024). *Understanding Merge Join Efficiency*. [https://learn.microsoft.com/en-us/sql/relational-databases/query-processing-architecture-guide](https://learn.microsoft.com/en-us/sql/relational-databases/query-processing-architecture-guide)

---

*Previous chapter: [Chapter 4 — Functions: String, Date, and NULL Handling](../chapter-04-functions/README.md)*

*Next chapter: [Chapter 6 — Aggregation: Summarising Data for Analysis](../chapter-06-aggregation/README.md)*

---

> **SQL for Analysts** | © Patrick Dolinger, NSCC Institute of Technology
> [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — Share and adapt freely with attribution
