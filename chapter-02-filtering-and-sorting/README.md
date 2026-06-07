# Chapter 2: Filtering and Sorting — Asking Precise Questions

> **SQL for Analysts**
> *A practical introduction to querying, transforming, and understanding relational data*
>
> © Patrick Dolinger, NSCC Institute of Technology
> Licensed under [Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/)
> You are free to share and adapt this material for any purpose, provided appropriate credit is given.

---

## Chapter Overview

Chapter 1 taught you how to retrieve data from a table. But retrieving *all* rows is rarely useful. A table with 16,000 rows returns 16,000 rows — and most of the time, you want to answer a specific question about a subset of them.

This chapter is about precision. The `WHERE` clause is the mechanism that turns a broad retrieval into a specific question. Combined with logical operators, pattern matching, range tests, and NULL handling, it gives you the tools to ask exactly the question you mean.

By the end of this chapter you will be able to:

- Write `WHERE` clauses using all standard comparison operators
- Combine multiple conditions using `AND`, `OR`, and `NOT`
- Use brackets to control the order of evaluation in compound conditions
- Filter rows using `IN`, `BETWEEN`, and `LIKE`
- Handle NULL values correctly using `IS NULL` and `IS NOT NULL`
- Remove duplicate rows from results using `DISTINCT`
- Combine filtering and sorting to produce precise, ordered results
- Predict the results of compound conditions without running the query

---

## Table of Contents

1. [The WHERE Clause: Filtering Rows](#1-the-where-clause-filtering-rows)
2. [Comparison Operators](#2-comparison-operators)
3. [Logical Operators: AND, OR, NOT](#3-logical-operators-and-or-not)
4. [Operator Precedence and Brackets](#4-operator-precedence-and-brackets)
5. [The IN Operator](#5-the-in-operator)
6. [The BETWEEN Operator](#6-the-between-operator)
7. [Pattern Matching with LIKE](#7-pattern-matching-with-like)
8. [NULL Values and IS NULL](#8-null-values-and-is-null)
9. [Removing Duplicates with DISTINCT](#9-removing-duplicates-with-distinct)
10. [Putting It Together: WHERE, ORDER BY, and TOP](#10-putting-it-together-where-order-by-and-top)
11. [Chapter Summary](#11-chapter-summary)
12. [Review Questions](#12-review-questions)
13. [🔍 Deeper Dive](#-deeper-dive)

---

## 1. The WHERE Clause: Filtering Rows

The `WHERE` clause specifies a condition that each row must satisfy to be included in the result. Rows that satisfy the condition are returned; rows that do not are excluded.

### 1.1 Position in the Query

`WHERE` always comes after `FROM` and before `ORDER BY`:

```sql
SELECT  column1, column2
FROM    schema.TableName
WHERE   condition
ORDER BY column1;
```

This sequence is the **SQL clause order** — a rule, not a convention. Writing clauses in the wrong order produces a syntax error.

### 1.2 A First Example

```sql
-- Return only products in the Sleeping category
USE CabotTrailOutdoorsSales;

SELECT  ProductName,
        CategoryName,
        UnitPrice
FROM    dim.Product
WHERE   CategoryName = 'Sleeping';
```

Run this. You should see 9 or 10 rows — only products whose `CategoryName` is exactly 'Sleeping'. The other 130+ products are excluded.

**Observation:** The string value `'Sleeping'` is wrapped in single quotes. In SQL, text values are always single-quoted. Numbers are not:

```sql
-- Text value: single quotes required
WHERE   CategoryName = 'Sleeping'

-- Numeric value: no quotes
WHERE   UnitPrice = 249.99
```

Putting quotes around a number (`WHERE UnitPrice = '249.99'`) may still work in SQL Server due to implicit type conversion, but it is imprecise and can cause unexpected behaviour. Match the data type: quotes for text, no quotes for numbers.

### 1.3 The WHERE Clause Processes Every Row

It helps to picture what SQL Server does when it executes a query with a `WHERE` clause:

1. Open the table (`FROM dim.Product`)
2. Read each row one at a time
3. Evaluate the `WHERE` condition for that row
4. If TRUE → include the row in the result
5. If FALSE or NULL → exclude the row
6. After all rows are processed, apply `ORDER BY` and `TOP` if present

This mental model — the database checking each row against the condition — helps you reason about what a complex `WHERE` clause will do without running it.

---

## 2. Comparison Operators

A comparison operator compares two values and returns TRUE, FALSE, or (in the case of NULLs) a special value called UNKNOWN. The `WHERE` clause includes rows where the comparison evaluates to TRUE.

### 2.1 The Standard Comparison Operators

| Operator | Meaning | Example |
|---|---|---|
| `=` | Equal to | `WHERE CategoryName = 'Sleeping'` |
| `<>` | Not equal to | `WHERE CategoryName <> 'Sleeping'` |
| `!=` | Not equal to (same as `<>`) | `WHERE CategoryName != 'Sleeping'` |
| `>` | Greater than | `WHERE UnitPrice > 100` |
| `<` | Less than | `WHERE UnitPrice < 50` |
| `>=` | Greater than or equal to | `WHERE UnitPrice >= 100` |
| `<=` | Less than or equal to | `WHERE UnitPrice <= 50` |

> **Style note:** `<>` is the SQL standard for "not equal to." `!=` works in SQL Server but is not part of the standard. Most experienced SQL developers use `<>`. Either is correct.

### 2.2 Comparing Strings

String comparisons in SQL Server are case-insensitive by default (see Deeper Dive for details). This means:

```sql
-- These three conditions return the same rows
WHERE CategoryName = 'Sleeping'
WHERE CategoryName = 'sleeping'
WHERE CategoryName = 'SLEEPING'
```

String comparisons are also alphabetical — 'A' < 'B', 'Apple' < 'Banana':

```sql
-- Products whose name comes alphabetically before 'D'
SELECT  ProductName
FROM    dim.Product
WHERE   ProductName < 'D'
ORDER BY ProductName;
```

This returns products whose names start with A, B, or C — 'Cape Breton' comes before 'D', so it is included.

### 2.3 Comparing Dates

Date comparisons work intuitively — later dates are "greater than" earlier dates:

```sql
-- Employees hired after January 1, 2023
SELECT  FullName,
        HireDate
FROM    dim.Employee
WHERE   HireDate > '2023-01-01'
ORDER BY HireDate;
```

Date literals in SQL Server are written as strings in quotes, in `'YYYY-MM-DD'` format. SQL Server converts them to the DATE data type automatically. Other formats (`'01/01/2023'`, `'Jan 1 2023'`) also work but are ambiguous across regional settings — always use `'YYYY-MM-DD'` for clarity and reliability.

### 2.4 Negation with `<>`

`<>` is frequently overlooked as a tool for "everything except." It is often cleaner than listing what you *do* want:

```sql
-- All categories except Shelter
SELECT  ProductName,
        CategoryName,
        UnitPrice
FROM    dim.Product
WHERE   CategoryName <> 'Shelter'
ORDER BY CategoryName, ProductName;
```

This returns products from all 12 other categories — far more concise than listing 12 category names explicitly.

---

## 3. Logical Operators: AND, OR, NOT

Real analytical questions often have multiple conditions. Logical operators combine them.

### 3.1 AND: Both Conditions Must Be True

`AND` returns rows where *all* conditions are TRUE:

```sql
-- Sleeping products that cost more than $200
SELECT  ProductName,
        CategoryName,
        UnitPrice
FROM    dim.Product
WHERE   CategoryName = 'Sleeping'
AND     UnitPrice    > 200
ORDER BY UnitPrice DESC;
```

Only rows where `CategoryName = 'Sleeping'` is TRUE *and* `UnitPrice > 200` is TRUE are included. A sleeping bag at $150 is excluded (price condition fails). A tent at $350 is excluded (category condition fails). Only sleeping bags over $200 qualify.

You can chain as many `AND` conditions as needed:

```sql
-- Active products in the Sleeping category priced between $100 and $300
SELECT  ProductName,
        UnitPrice,
        IsDiscontinued
FROM    dim.Product
WHERE   CategoryName    = 'Sleeping'
AND     UnitPrice       >= 100
AND     UnitPrice       <= 300
AND     IsDiscontinued  = 0
ORDER BY UnitPrice;
```

### 3.2 OR: At Least One Condition Must Be True

`OR` returns rows where *at least one* condition is TRUE:

```sql
-- Products in either the Sleeping or Shelter category
SELECT  ProductName,
        CategoryName,
        UnitPrice
FROM    dim.Product
WHERE   CategoryName = 'Sleeping'
OR      CategoryName = 'Shelter'
ORDER BY CategoryName, UnitPrice;
```

A product qualifies if its category is 'Sleeping' *or* 'Shelter' (or both — though a product cannot be in both simultaneously since `CategoryName` holds one value per row).

`OR` is more permissive than `AND`. Adding more `OR` conditions expands the result set; adding more `AND` conditions narrows it.

### 3.3 NOT: Inverting a Condition

`NOT` reverses the TRUE/FALSE result of a condition:

```sql
-- All products that are NOT discontinued
SELECT  ProductName,
        CategoryName,
        IsDiscontinued
FROM    dim.Product
WHERE   NOT IsDiscontinued = 1
ORDER BY CategoryName, ProductName;
```

This is equivalent to `WHERE IsDiscontinued = 0` — but `NOT` is useful when the condition being negated is complex. It also has specific uses with `IN`, `BETWEEN`, and `LIKE` that you will see shortly.

---

## 4. Operator Precedence and Brackets

### 4.1 The Precedence Problem

SQL evaluates `AND` before `OR` — just as multiplication is evaluated before addition in arithmetic. This means:

```sql
WHERE CategoryName = 'Sleeping' OR CategoryName = 'Shelter' AND UnitPrice > 200
```

is evaluated as:

```sql
WHERE CategoryName = 'Sleeping' OR (CategoryName = 'Shelter' AND UnitPrice > 200)
```

**Not** as:

```sql
WHERE (CategoryName = 'Sleeping' OR CategoryName = 'Shelter') AND UnitPrice > 200
```

The first interpretation returns *all* sleeping products (any price) plus expensive shelter products. The second returns only sleeping and shelter products that cost more than $200. These produce very different results.

### 4.2 Always Use Brackets to Make Intent Clear

When you mix `AND` and `OR`, use brackets to make the evaluation order explicit — not to rely on precedence rules that you and the next developer may both misremember:

```sql
-- CLEAR: brackets show exactly what is intended
-- Sleeping OR Shelter, but only if price > $200
SELECT  ProductName,
        CategoryName,
        UnitPrice
FROM    dim.Product
WHERE   (CategoryName = 'Sleeping' OR CategoryName = 'Shelter')
AND     UnitPrice > 200
ORDER BY CategoryName, UnitPrice;
```

Compare to the ambiguous version without brackets:

```sql
-- AMBIGUOUS: easy to misread
WHERE   CategoryName = 'Sleeping'
OR      CategoryName = 'Shelter'
AND     UnitPrice > 200
```

Both SQL Server and human readers will parse these differently. The bracketed version leaves no doubt.

**Rule:** Whenever you mix `AND` and `OR` in the same `WHERE` clause, use brackets. No exceptions.

### 4.3 A Precedence Test

Before running this query, predict what it returns:

```sql
SELECT  ProductName, CategoryName, UnitPrice
FROM    dim.Product
WHERE   CategoryName = 'Sleeping'
OR      CategoryName = 'Accessories'
AND     UnitPrice < 30;
```

Due to `AND` before `OR`, this is:
```sql
WHERE CategoryName = 'Sleeping'
OR   (CategoryName = 'Accessories' AND UnitPrice < 30)
```

It returns *all* sleeping products (regardless of price) plus accessories under $30. Is that what you wanted? Probably not — you likely wanted sleeping or accessories products, both under $30. The corrected query:

```sql
WHERE   (CategoryName = 'Sleeping' OR CategoryName = 'Accessories')
AND     UnitPrice < 30;
```

Run both versions and compare the row counts. The difference will be instructive.

---

## 5. The IN Operator

### 5.1 What IN Does

`IN` tests whether a value matches any value in a list. It is shorthand for multiple `OR` conditions on the same column:

```sql
-- Without IN: verbose
WHERE   CategoryName = 'Sleeping'
OR      CategoryName = 'Shelter'
OR      CategoryName = 'Footwear'
OR      CategoryName = 'Apparel - Tops'

-- With IN: concise
WHERE   CategoryName IN ('Sleeping', 'Shelter', 'Footwear', 'Apparel - Tops')
```

Both produce identical results. `IN` is not faster — it compiles to the same execution plan. But it is dramatically more readable when the list has more than two values.

### 5.2 IN with a List of Values

```sql
-- Customers in Atlantic provinces
SELECT  CustomerName,
        ProvinceCode,
        SalesTerritory
FROM    dim.Customer
WHERE   ProvinceCode IN ('NS', 'NB', 'PE', 'NL')
ORDER BY ProvinceCode, CustomerName;
```

```sql
-- Products in high-margin categories
SELECT  ProductName,
        CategoryName,
        UnitPrice
FROM    dim.Product
WHERE   CategoryName IN (
    'Accessories',
    'Hydration',
    'Navigation',
    'Safety and First Aid'
)
ORDER BY CategoryName, UnitPrice DESC;
```

The list can be written on one line or spread across multiple lines. When lists are long, spreading across lines with consistent indentation makes them scannable.

### 5.3 NOT IN

`NOT IN` returns rows where the value does *not* match any value in the list:

```sql
-- Products NOT in gear-intensive categories
SELECT  ProductName,
        CategoryName
FROM    dim.Product
WHERE   CategoryName NOT IN ('Climbing and Rappelling', 'Paddling', 'Winter Sports')
ORDER BY CategoryName, ProductName;
```

> **Important warning about NOT IN and NULLs:** `NOT IN` behaves unexpectedly when the list contains a NULL value. If any value in the list is NULL, `NOT IN` returns no rows — because `value <> NULL` evaluates to UNKNOWN, not FALSE. This is a subtle but significant trap. When using `NOT IN` with a subquery that might return NULLs, use `NOT EXISTS` instead (covered in Chapter 7).

---

## 6. The BETWEEN Operator

### 6.1 What BETWEEN Does

`BETWEEN` tests whether a value falls within a range, *inclusive of both endpoints*:

```sql
-- Products priced between $50 and $150
SELECT  ProductName,
        CategoryName,
        UnitPrice
FROM    dim.Product
WHERE   UnitPrice BETWEEN 50 AND 150
ORDER BY UnitPrice;
```

This includes products at exactly $50, at exactly $150, and everything in between. It is equivalent to:

```sql
WHERE UnitPrice >= 50 AND UnitPrice <= 150
```

The `BETWEEN` version is more readable for range conditions.

### 6.2 BETWEEN with Dates

`BETWEEN` works on dates as naturally as on numbers:

```sql
-- Calendar dates in Q3 2024 (July, August, September)
SELECT  FullDate,
        MonthName,
        DayName,
        IsWeekend
FROM    dim.Calendar
WHERE   FullDate BETWEEN '2024-07-01' AND '2024-09-30'
ORDER BY FullDate;
```

> **Inclusive endpoints matter:** `'2024-09-30'` includes September 30th. If the column were a DATETIME rather than a DATE, `'2024-09-30'` would be interpreted as `2024-09-30 00:00:00` — which would exclude transactions that occurred on September 30th after midnight. For DATETIME columns, use `< '2024-10-01'` rather than `<= '2024-09-30'` to avoid this trap. Since `dim.Calendar.FullDate` is a DATE column in CabotTrail, `BETWEEN` works correctly here.

### 6.3 NOT BETWEEN

`NOT BETWEEN` returns rows outside the range:

```sql
-- Products outside the mid-range price band
SELECT  ProductName,
        CategoryName,
        UnitPrice,
        CASE
            WHEN UnitPrice < 50  THEN 'Budget'
            WHEN UnitPrice > 300 THEN 'Premium'
        END AS PriceBand
FROM    dim.Product
WHERE   UnitPrice NOT BETWEEN 50 AND 300
ORDER BY UnitPrice;
```

This returns products under $50 or over $300 — the extremes of the price range.

---

## 7. Pattern Matching with LIKE

### 7.1 What LIKE Does

`LIKE` tests whether a string value matches a pattern. It is the tool for "contains," "starts with," and "ends with" searches that exact equality (`=`) cannot express.

Two wildcard characters build the pattern:

| Wildcard | Matches | Example |
|---|---|---|
| `%` | Any sequence of zero or more characters | `'%Trail%'` matches 'Cabot Trail Tent', 'TrailBlazer', 'On the Trail' |
| `_` | Exactly one character | `'CT_001'` matches 'CT-001', 'CT0001', 'CTA001' |

### 7.2 Common LIKE Patterns

**Starts with:**
```sql
-- Products whose name starts with 'Cabot'
SELECT  ProductName, CategoryName
FROM    dim.Product
WHERE   ProductName LIKE 'Cabot%'
ORDER BY ProductName;
```

**Ends with:**
```sql
-- Products whose name ends with 'Kit'
SELECT  ProductName, CategoryName
FROM    dim.Product
WHERE   ProductName LIKE '%Kit'
ORDER BY ProductName;
```

**Contains:**
```sql
-- Products that contain 'Trail' anywhere in the name
SELECT  ProductName, CategoryName, UnitPrice
FROM    dim.Product
WHERE   ProductName LIKE '%Trail%'
ORDER BY ProductName;
```

**Exactly one character wildcard:**
```sql
-- Product codes matching a specific pattern: two letters, a dash, three digits
SELECT  ProductCode, ProductName
FROM    dim.Product
WHERE   ProductCode LIKE '__-___'
ORDER BY ProductCode;
-- Two underscores = two any characters, dash is literal, three underscores = three any chars
```

### 7.3 NOT LIKE

`NOT LIKE` returns rows where the pattern does not match:

```sql
-- Products whose name does NOT contain 'Cabot'
SELECT  ProductName, CategoryName
FROM    dim.Product
WHERE   ProductName NOT LIKE '%Cabot%'
ORDER BY ProductName;
```

### 7.4 LIKE Performance Considerations

`LIKE 'Cabot%'` (pattern anchored at the start) can use a database index efficiently — SQL Server knows where to start looking alphabetically. `LIKE '%Trail%'` (pattern with a leading `%`) cannot use an index — every row must be examined. On small dimension tables, this is imperceptible. On large fact tables with millions of rows, leading wildcards can be very slow.

For analytical queries against the CabotTrail dimension tables (142 products, 100 customers), LIKE performance is not a concern. Keep this limitation in mind as you work with larger datasets.

---

## 8. NULL Values and IS NULL

### 8.1 What NULL Means

`NULL` is SQL's way of representing "no value" or "unknown." It is not zero, it is not an empty string, it is not the word "null" — it is the complete absence of a value.

NULL appears in columns where information is optional or simply not available. In CabotTrail's `dim.Product` table, the `ProductCode` column is nullable — some products have a code, others do not:

```sql
-- See which products have no product code
SELECT  ProductName,
        ProductCode,
        CategoryName
FROM    dim.Product
WHERE   ProductCode IS NULL
ORDER BY CategoryName, ProductName;
```

### 8.2 Why You Cannot Use = NULL

The most common NULL mistake is trying to compare with `=`:

```sql
-- WRONG: this returns no rows in SQL Server
WHERE ProductCode = NULL

-- CORRECT: use IS NULL
WHERE ProductCode IS NULL
```

The reason: NULL represents an unknown value. Comparing an unknown to anything — even another NULL — produces UNKNOWN, not TRUE. `NULL = NULL` is UNKNOWN, not TRUE. SQL Server therefore correctly excludes all rows from `WHERE ProductCode = NULL`.

This is counterintuitive but consistent with the mathematical semantics of NULL as "unknown." It is one of SQL's most important rules to internalize early.

### 8.3 IS NULL and IS NOT NULL

```sql
-- Products WITH a product code
SELECT  ProductName,
        ProductCode,
        CategoryName
FROM    dim.Product
WHERE   ProductCode IS NOT NULL
ORDER BY ProductCode;
```

```sql
-- Employees with no email address on record
SELECT  FullName,
        JobTitle,
        EmailAddress
FROM    dim.Employee
WHERE   EmailAddress IS NULL
ORDER BY FullName;
```

### 8.4 NULL in Comparisons and Calculations

NULL propagates through comparisons and calculations:

```sql
-- Any comparison involving NULL returns UNKNOWN
NULL = 5        → UNKNOWN
NULL <> 5       → UNKNOWN
NULL > 5        → UNKNOWN
NULL = NULL     → UNKNOWN

-- Any arithmetic involving NULL returns NULL
NULL + 5        → NULL
NULL * 100      → NULL
10 / NULL       → NULL
```

This means: if a column has NULL values, any `WHERE` clause that compares that column with `=`, `<>`, `>`, `<`, etc. will silently exclude the NULL rows — because the comparison produces UNKNOWN, not TRUE.

```sql
-- This does NOT return rows where UnitPrice is NULL
WHERE UnitPrice > 0

-- To also catch NULL prices, you need:
WHERE UnitPrice > 0 OR UnitPrice IS NULL
```

Remembering that NULL propagates through comparisons prevents a whole category of subtle bugs.

---

## 9. Removing Duplicates with DISTINCT

### 9.1 What DISTINCT Does

`DISTINCT` removes duplicate rows from the result set. Rows are considered duplicate if every column in the SELECT list has the same value.

```sql
-- What distinct categories exist in the product table?
SELECT  DISTINCT CategoryName
FROM    dim.Product
ORDER BY CategoryName;
```

Without `DISTINCT`, this returns 142 rows — one per product, with many repeated category names. With `DISTINCT`, it returns 13 rows — one per unique category.

### 9.2 DISTINCT on Multiple Columns

When applied to multiple columns, `DISTINCT` removes rows where the *combination* of all selected columns is identical:

```sql
-- What distinct province/territory combinations exist in the customer table?
SELECT  DISTINCT
        ProvinceCode,
        SalesTerritory
FROM    dim.Customer
ORDER BY SalesTerritory, ProvinceCode;
```

This returns one row per unique (ProvinceCode, SalesTerritory) pair — not one per unique province alone or territory alone.

### 9.3 DISTINCT vs GROUP BY

`DISTINCT` and `GROUP BY` both eliminate duplicates. For simple cases (no aggregation needed), they produce the same result:

```sql
-- These return identical results
SELECT DISTINCT CategoryName FROM dim.Product ORDER BY CategoryName;
SELECT CategoryName FROM dim.Product GROUP BY CategoryName ORDER BY CategoryName;
```

`GROUP BY` is more powerful — it allows aggregation (counting how many products are in each category, for example). `DISTINCT` is simpler when you only need the unique values and no aggregation. Chapter 6 covers `GROUP BY` in depth.

### 9.4 When DISTINCT Is the Wrong Tool

`DISTINCT` is sometimes used to "fix" a query that is returning unexpected duplicates. If your query produces duplicate rows that you did not expect, the duplicates are usually a symptom of an incorrect `JOIN` — a join condition that matches more rows than you intended. Adding `DISTINCT` hides the symptom without fixing the cause. It is better to understand why the duplicates exist and fix the query logic.

You will encounter this issue in Chapter 5 when learning JOINs. For now, the takeaway: use `DISTINCT` intentionally, not as a band-aid.

---

## 10. Putting It Together: WHERE, ORDER BY, and TOP

All the filtering tools covered in this chapter combine naturally with the `ORDER BY` and `TOP` from Chapter 1. The key is remembering the clause order:

```sql
SELECT   [TOP n] column1, column2
FROM     schema.TableName
WHERE    condition
ORDER BY column1 [ASC|DESC];
```

### 10.1 Business Questions Answered

Let's apply everything from this chapter to real analytical questions about CabotTrail.

**Q: What are the ten least expensive active products, and what categories are they in?**

```sql
SELECT  TOP 10
        ProductName,
        CategoryName,
        UnitPrice
FROM    dim.Product
WHERE   IsDiscontinued = 0
ORDER BY UnitPrice ASC;
```

**Q: Which customers in Ontario or British Columbia have a credit limit above $5,000?**

```sql
SELECT  CustomerName,
        ProvinceCode,
        SalesTerritory,
        CreditLimit
FROM    dim.Customer
WHERE   ProvinceCode IN ('ON', 'BC')
AND     CreditLimit > 5000
ORDER BY CreditLimit DESC;
```

**Q: Which calendar dates in 2024 are public holidays that fall on a weekday?**

```sql
SELECT  FullDate,
        DayName,
        HolidayName,
        IsWeekend
FROM    dim.Calendar
WHERE   CalendarYear    = 2024
AND     IsHoliday       = 1
AND     IsWeekend       = 0
ORDER BY FullDate;
```

**Q: Which products have 'Cabot' in their name and cost between $100 and $500?**

```sql
SELECT  ProductName,
        CategoryName,
        UnitPrice
FROM    dim.Product
WHERE   ProductName LIKE '%Cabot%'
AND     UnitPrice BETWEEN 100 AND 500
ORDER BY UnitPrice;
```

**Q: Which employees have a NULL email address or work in the 'Operations' department?**

```sql
SELECT  FullName,
        Department,
        EmailAddress
FROM    dim.Employee
WHERE   EmailAddress IS NULL
OR      Department = 'Operations'
ORDER BY Department, FullName;
```

### 10.2 Reading Queries Backwards

A useful technique for understanding complex `WHERE` clauses: read them from the result backward to the conditions.

Given:
```sql
SELECT  ProductName, CategoryName, UnitPrice
FROM    dim.Product
WHERE   (CategoryName IN ('Sleeping', 'Shelter') OR UnitPrice > 400)
AND     IsDiscontinued = 0
ORDER BY UnitPrice DESC;
```

Reading backward: "I want active products (IsDiscontinued = 0) that are either in Sleeping/Shelter OR cost more than $400."

This mental habit helps you predict results without running a query and catch logic errors before they happen.

---

## 11. Chapter Summary

- The `WHERE` clause filters rows by specifying conditions. Rows where the condition evaluates to TRUE are returned; FALSE and UNKNOWN rows are excluded.

- **Comparison operators** (`=`, `<>`, `>`, `<`, `>=`, `<=`) compare column values to literals. Strings use single quotes; numbers and NULLs do not.

- **AND** returns rows where all conditions are TRUE. **OR** returns rows where at least one condition is TRUE. **NOT** reverses a condition. `AND` has higher precedence than `OR`.

- **Always use brackets** when mixing `AND` and `OR`. Without them, queries are ambiguous and easy to misread.

- **IN** is shorthand for multiple `OR` conditions on the same column. **NOT IN** excludes matching values (but use with caution when NULLs may be present).

- **BETWEEN** tests inclusive ranges. For DATETIME columns, prefer `>= start AND < end_plus_one_day` to avoid boundary issues.

- **LIKE** matches string patterns using `%` (any characters) and `_` (exactly one character). Leading `%` prevents index use on large tables.

- **NULL** means "no value" — it is not zero or empty string. Use `IS NULL` and `IS NOT NULL`, never `= NULL`. NULL propagates through comparisons and arithmetic, producing UNKNOWN or NULL respectively.

- **DISTINCT** removes duplicate rows where all selected columns are identical. Use it intentionally; do not use it to mask JOIN errors.

---

## 12. Review Questions

1. Explain why `WHERE ProductCode = NULL` returns no rows in SQL Server. What should you write instead, and what is the mathematical reason the equals operator does not work with NULL?

2. Predict what the following query returns *without running it*, then verify by running it. Explain the difference between the intended question and what the query actually asks:
   ```sql
   SELECT ProductName, CategoryName, UnitPrice
   FROM   dim.Product
   WHERE  CategoryName = 'Sleeping'
   OR     CategoryName = 'Footwear'
   AND    UnitPrice > 150;
   ```

3. Write a query that returns all customers from Nova Scotia (`ProvinceCode = 'NS'`) or New Brunswick (`ProvinceCode = 'NB'`) who have a credit limit between $2,000 and $8,000 (inclusive). Include their name, province code, sales territory, and credit limit. Sort by credit limit descending.

4. Write a query that returns all products where `ProductCode` is NOT NULL and the product name contains the word 'Trail'. Include the product code, product name, and category. Sort alphabetically by product name.

5. The following query is meant to return "products in the Sleeping or Shelter category that cost less than $200." Identify the logic error, explain what the query actually returns, and rewrite it correctly:
   ```sql
   SELECT ProductName, CategoryName, UnitPrice
   FROM   dim.Product
   WHERE  CategoryName = 'Sleeping'
   OR     CategoryName = 'Shelter'
   AND    UnitPrice < 200;
   ```

6. Write a query against `dim.Calendar` that returns all dates in the first quarter of fiscal year 2025 (`FiscalYearNumber = 2025` and `FiscalQuarterNumber = 1`). Include `FullDate`, `DayName`, `MonthName`, and `IsHoliday`. Sort chronologically.

7. Write a query that returns all employees whose `Department` is NOT IN the list `('IT', 'Legal', 'Executive')`. Include their full name, job title, and department. Sort by department, then by full name.

8. A colleague writes this query to find products that have either no product code or a price under $25:
   ```sql
   SELECT ProductName, ProductCode, UnitPrice
   FROM   dim.Product
   WHERE  ProductCode = NULL OR UnitPrice < 25;
   ```
   The query runs without error but the results are wrong — products with NULL product codes are missing. Explain the bug and write the corrected query.

---

## 🔍 Deeper Dive

### Going Further with Filtering and Sorting

#### The Three-Valued Logic of SQL

SQL does not use simple true/false (Boolean) logic. It uses **three-valued logic** with TRUE, FALSE, and UNKNOWN. UNKNOWN arises whenever NULL is involved in a comparison.

The rules of three-valued logic:

| Expression | Result |
|---|---|
| `TRUE AND UNKNOWN` | UNKNOWN |
| `FALSE AND UNKNOWN` | FALSE |
| `TRUE OR UNKNOWN` | TRUE |
| `FALSE OR UNKNOWN` | UNKNOWN |
| `NOT UNKNOWN` | UNKNOWN |

These rules have practical consequences. `FALSE AND UNKNOWN` is FALSE (because AND requires both to be TRUE — if one is FALSE, it does not matter what the other is). `TRUE OR UNKNOWN` is TRUE (because OR needs only one TRUE). But `TRUE AND UNKNOWN` is UNKNOWN — and rows with UNKNOWN conditions are excluded from results.

This is why NULL comparisons are subtle: they can produce UNKNOWN in ways that look like they should produce TRUE, silently excluding rows from your result set.

The `WHERE` clause includes only TRUE rows. Both FALSE and UNKNOWN rows are excluded. This asymmetry between FALSE and UNKNOWN matters for `NOT` — `NOT FALSE = TRUE`, but `NOT UNKNOWN = UNKNOWN`. Both exclude the row, but for different reasons.

Understanding three-valued logic explains many surprising SQL behaviours. For a thorough treatment:
Date, C. J. (2011). *SQL and Relational Theory: How to Write Accurate SQL Code* (2nd ed.). O'Reilly Media. — Chapter 4 covers NULL and three-valued logic in depth.

#### Case Sensitivity and Collations

SQL Server's default collation (`SQL_Latin1_General_CP1_CI_AS`) is **case-insensitive** (`CI`). String comparisons treat 'A' and 'a' as equal. This is convenient for most business queries but means you cannot distinguish uppercase from lowercase.

If you need case-sensitive comparison, you can force it using the `COLLATE` clause:

```sql
-- Case-sensitive comparison (returns only exact uppercase match)
SELECT ProductName
FROM   dim.Product
WHERE  ProductName COLLATE Latin1_General_CS_AS = 'Cape Breton Sleeping Bag';
-- CS = Case Sensitive
```

Collations also affect sort order. `CI_AS` sorts case-insensitively: 'apple', 'Apple', and 'APPLE' sort together. A case-sensitive collation would sort them separately.

Database administrators set the collation at the server, database, or column level. As an analyst, you usually work within the default — but knowing collations exist explains why sometimes a query on a different database seems to behave differently with string comparisons.

Microsoft documentation on collations:
[Collation and Unicode support — SQL Server](https://learn.microsoft.com/en-us/sql/relational-databases/collations/collation-and-unicode-support)

#### LIKE Escape Characters

What if you need to search for a literal `%` or `_` character in a string? These are wildcards in LIKE patterns — how do you match them literally?

Use the `ESCAPE` clause to define an escape character:

```sql
-- Find product codes containing a literal underscore
SELECT ProductCode, ProductName
FROM   dim.Product
WHERE  ProductCode LIKE '%!_%' ESCAPE '!';
-- The ! before _ tells SQL Server: the next character is literal, not a wildcard
```

In this example, `!` is the escape character. `!_` means "a literal underscore." Any character can be the escape character — `!`, `\`, or `~` are common choices since they rarely appear in data.

This is an edge case in most analytical work — but if you are ever querying financial data, product codes, or file paths that contain underscores or percent signs, you will need it.

#### Regular Expressions in Other Databases

SQL Server's `LIKE` operator is limited to `%` and `_` wildcards. Other database systems — PostgreSQL, MySQL, BigQuery, Snowflake — support **regular expressions** for pattern matching, which are far more powerful:

```sql
-- PostgreSQL: find products matching a regex pattern
WHERE ProductName ~ '^Cabot.*Bag$'
-- Matches: starts with 'Cabot', ends with 'Bag'

-- SQL Server equivalent (no native regex):
WHERE ProductName LIKE 'Cabot%Bag'
-- Less precise: does not require 'Bag' to be at the very end
```

Regular expressions can match complex patterns — digits, letters, specific character classes, exact repetitions — that `LIKE` cannot express. If you work with PostgreSQL or cloud databases in future roles, learning regular expression syntax opens powerful filtering capabilities.

#### Sargability: Writing WHERE Clauses That Use Indexes

The term **sargable** (Search ARGument ABLE) describes a `WHERE` condition that can use a database index efficiently. Non-sargable conditions force SQL Server to scan every row in the table.

**Sargable patterns (fast on large tables):**
```sql
WHERE OrderDate >= '2024-01-01'                  -- direct column comparison
WHERE CustomerName = 'Fundy Bay Outfitters'      -- direct column comparison
WHERE UnitPrice BETWEEN 50 AND 150               -- range on column directly
WHERE ProductCode LIKE 'CT%'                     -- anchored-start LIKE
```

**Non-sargable patterns (slow on large tables):**
```sql
WHERE YEAR(OrderDate) = 2024                     -- function on column prevents index
WHERE UPPER(CustomerName) = 'FUNDY BAY'         -- function on column
WHERE ProductName LIKE '%Trail%'                 -- leading % prevents index
WHERE UnitPrice * 1.15 > 200                     -- expression on column
```

The general rule: **apply functions to the literal, not the column.** Instead of `WHERE YEAR(OrderDate) = 2024`, use `WHERE OrderDate >= '2024-01-01' AND OrderDate < '2025-01-01'`. The first requires SQL Server to compute `YEAR()` for every row; the second lets it find qualifying rows directly using an index.

For the CabotTrail dimension tables (100–142 rows), sargability is irrelevant — the tables are tiny. For large fact tables and in production work, writing sargable conditions is a fundamental performance skill.

---

### Industry Perspectives

#### The WHERE Clause as a Business Rule

Every `WHERE` clause you write encodes a business rule. `WHERE IsDiscontinued = 0` encodes "we only analyse active products." `WHERE CalendarYear >= 2022` encodes "we only look at data from 2022 onward." `WHERE ProvinceCode IN ('NS', 'NB', 'PE', 'NL')` encodes "Atlantic Canada includes these four provinces."

Business rules encoded in SQL queries are invisible unless documented. The next analyst who uses your query will not know *why* those filters were applied — was it a technical requirement, a business decision, or a mistake?

Good commenting practice (from Chapter 1) is essential here: document not just *what* a filter does but *why* it exists. "Filter to active products per business requirement dated 2024-03-01" is far more informative than no comment at all.

#### NULL in Practice: The Most Common Source of Subtle Bugs

In surveys of experienced SQL developers, NULL-related bugs consistently rank as the most common source of query errors that produce wrong results without errors being raised. The query runs; the results look plausible; but they are missing rows due to NULL propagation.

The most common manifestations:
- `WHERE column = NULL` (should be `IS NULL`)
- Forgetting that `NOT IN` with NULLs returns nothing
- Arithmetic on nullable columns producing NULL totals
- Aggregations (`SUM`, `AVG`) silently ignoring NULL rows

The discipline of checking for NULLs before writing `WHERE` conditions — using `INFORMATION_SCHEMA.COLUMNS` to see which columns are nullable — is a professional habit that prevents this class of error.

---

### References and Further Reading

1. Date, C. J. (2011). *SQL and Relational Theory: How to Write Accurate SQL Code* (2nd ed.). O'Reilly Media. — Chapter 4 covers NULL and three-valued logic more rigorously than any other introductory text.

2. Microsoft. (2024). *WHERE (Transact-SQL)*. [https://learn.microsoft.com/en-us/sql/t-sql/queries/where-transact-sql](https://learn.microsoft.com/en-us/sql/t-sql/queries/where-transact-sql)

3. Microsoft. (2024). *LIKE (Transact-SQL)*. [https://learn.microsoft.com/en-us/sql/t-sql/language-elements/like-transact-sql](https://learn.microsoft.com/en-us/sql/t-sql/language-elements/like-transact-sql)

4. Microsoft. (2024). *NULL and UNKNOWN (Transact-SQL)*. [https://learn.microsoft.com/en-us/sql/t-sql/language-elements/null-and-unknown-transact-sql](https://learn.microsoft.com/en-us/sql/t-sql/language-elements/null-and-unknown-transact-sql)

5. Microsoft. (2024). *Collation and Unicode support*. [https://learn.microsoft.com/en-us/sql/relational-databases/collations/collation-and-unicode-support](https://learn.microsoft.com/en-us/sql/relational-databases/collations/collation-and-unicode-support)

6. Fritchey, G. (2018). *SQL Server Query Performance Tuning* (5th ed.). Apress. — Chapter 3 covers sargability and index-friendly WHERE clauses in depth.

7. Ben-Gan, I. (2015). *T-SQL Fundamentals* (3rd ed.). Microsoft Press. — Chapter 2 provides an excellent treatment of filtering, NULL handling, and three-valued logic for SQL Server specifically.

---

*Previous chapter: [Chapter 1 — The Relational World: Databases, Tables, and Your First Query](../chapter-01-relational-world/README.md)*

*Next chapter: [Chapter 3 — Data Types, Math, and Calculated Columns](../chapter-03-data-types-and-math/README.md)*

---

> **SQL for Analysts** | © Patrick Dolinger, NSCC Institute of Technology
> [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — Share and adapt freely with attribution
