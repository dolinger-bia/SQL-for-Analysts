# Chapter 9: Moving Data — DDL, DML, and Source-to-Target Scripts

> **SQL for Analysts**
> *A practical introduction to querying, transforming, and understanding relational data*
>
> © Patrick Dolinger, NSCC Institute of Technology
> Licensed under [Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/)
> You are free to share and adapt this material for any purpose, provided appropriate credit is given.

---

## Chapter Overview

Every chapter up to this point has been about reading data — retrieving, filtering, joining, aggregating, and analysing rows that already exist. This final chapter crosses to the other side: creating the structures that hold data and moving data from one place to another.

This is where SQL for analysts meets SQL for data engineers. The skills here — designing a target table, writing a source-to-target INSERT, verifying the result — are the foundation of the ETL work you will do in DBAS 2103. They are also the skills you need as an analyst when you need to save a complex query's result for reuse, build a summary table for a report, or prototype a new analytical structure.

By the end of this chapter you will be able to:

- Write `CREATE TABLE` statements with appropriate data types and constraints
- Explain primary keys, foreign keys, and the constraints that enforce data integrity
- Write `ALTER TABLE` statements to modify existing structures
- Write `INSERT INTO ... VALUES` to add individual rows
- Write `INSERT INTO ... SELECT` to populate a table from a query
- Write `SELECT INTO` for rapid prototyping
- Write `UPDATE` and `DELETE` statements with the SELECT-first discipline
- Write a complete source-to-target script that creates a target, populates it, and verifies the result
- Explain the relationship between the T-SQL skills in this book and the SSIS ETL work in DBAS 2103

---

## Table of Contents

1. [DDL: Creating Database Structures](#1-ddl-creating-database-structures)
2. [CREATE TABLE: Defining a Target](#2-create-table-defining-a-target)
3. [Constraints: Enforcing Business Rules](#3-constraints-enforcing-business-rules)
4. [ALTER TABLE: Modifying Structures](#4-alter-table-modifying-structures)
5. [DROP TABLE: Removing Structures](#5-drop-table-removing-structures)
6. [DML Revisited: INSERT, UPDATE, DELETE](#6-dml-revisited-insert-update-delete)
7. [INSERT INTO … SELECT: Populating from a Query](#7-insert-into--select-populating-from-a-query)
8. [SELECT INTO: Rapid Prototyping](#8-select-into-rapid-prototyping)
9. [Writing a Source-to-Target Script](#9-writing-a-source-to-target-script)
10. [Verifying Your Work: Reconciliation Queries](#10-verifying-your-work-reconciliation-queries)
11. [From T-SQL Scripts to SSIS Packages](#11-from-t-sql-scripts-to-ssis-packages)
12. [Looking Back, Looking Forward](#12-looking-back-looking-forward)
13. [Chapter Summary](#13-chapter-summary)
14. [Review Questions](#14-review-questions)
15. [🔍 Deeper Dive](#-deeper-dive)

---

## 1. DDL: Creating Database Structures

In Chapter 3 you read data types from the system catalog. In this chapter you use them to define your own tables. **Data Definition Language (DDL)** is the SQL category that creates, modifies, and removes database objects.

### 1.1 The DDL Statement Family

| Statement | What it does |
|---|---|
| `CREATE TABLE` | Defines a new table with its columns and constraints |
| `ALTER TABLE` | Modifies an existing table (add/remove/change columns or constraints) |
| `DROP TABLE` | Permanently removes a table and all its data |
| `CREATE INDEX` | Creates an index to speed up queries |
| `DROP INDEX` | Removes an index |
| `TRUNCATE TABLE` | Removes all rows from a table (faster than DELETE, minimal logging) |

DDL changes the **structure** of the database. DML changes the **data** in that structure. The distinction matters for permissions — DBAs often restrict who can run DDL while allowing analysts broader DML access.

### 1.2 The Design-Before-Build Principle

Chapter 6 of the companion textbook *ETL for Business Intelligence* established the principle: **design before you build**. Define the target structure — column names, data types, constraints — before writing any INSERT statements. The reason: changing a table structure after data exists is possible but disruptive. Choosing the right data type at creation costs five minutes; migrating data to a different type after millions of rows exist costs days.

For this chapter's exercises, the design step is as important as the SQL step.

---

## 2. CREATE TABLE: Defining a Target

### 2.1 Basic Syntax

```sql
CREATE TABLE schema_name.table_name
(
    column1_name    data_type       [NULL | NOT NULL] [DEFAULT value],
    column2_name    data_type       [NULL | NOT NULL] [DEFAULT value],
    ...
    [CONSTRAINT constraint_name constraint_type (...)]
);
```

### 2.2 A First Table

```sql
-- Create a monthly sales summary table
USE CabotTrailOutdoorsSales;
GO

CREATE TABLE dbo.MonthlySalesSummary
(
    SummaryID       INT             NOT NULL IDENTITY(1,1),
    CalendarYear    SMALLINT        NOT NULL,
    MonthNumber     TINYINT         NOT NULL,
    MonthName       NVARCHAR(10)    NOT NULL,
    SalesTerritory  NVARCHAR(60)    NOT NULL,
    InvoiceCount    INT             NOT NULL    DEFAULT 0,
    SalesLineCount  INT             NOT NULL    DEFAULT 0,
    TotalRevenue    DECIMAL(18,2)   NOT NULL    DEFAULT 0,
    TotalGrossProfit DECIMAL(18,2)  NOT NULL    DEFAULT 0,
    AvgMarginPct    DECIMAL(8,2)    NULL,
    CreatedDate     DATE            NOT NULL    DEFAULT GETDATE()
);
```

Walk through each column:

- `SummaryID INT NOT NULL IDENTITY(1,1)` — auto-incrementing integer primary key candidate. `IDENTITY(1,1)` means start at 1, increment by 1.
- `CalendarYear SMALLINT NOT NULL` — year number. `SMALLINT` is sufficient (range ±32,767) and uses 2 bytes instead of 4.
- `MonthNumber TINYINT NOT NULL` — 1–12. `TINYINT` (0–255, 1 byte) is the right size.
- `SalesTerritory NVARCHAR(60) NOT NULL` — variable-length Unicode text.
- `TotalRevenue DECIMAL(18,2) NOT NULL DEFAULT 0` — exact decimal for financial amounts; default of 0 prevents NULL in a mandatory column.
- `AvgMarginPct DECIMAL(8,2) NULL` — nullable because it might not be computable (if revenue is 0).
- `CreatedDate DATE NOT NULL DEFAULT GETDATE()` — automatically populated with today's date when a row is inserted.

### 2.3 Choosing Data Types: The Design Decisions

Every column in the CREATE TABLE statement above reflects a design decision:

| Question | Answer in this table |
|---|---|
| Can this column be empty? | `AvgMarginPct` may be NULL (no sales); all others NOT NULL |
| What is the largest possible value? | SMALLINT for year, TINYINT for month number |
| Is Unicode needed? | Yes for territory name (`NVARCHAR`) |
| What precision is needed for financial values? | `DECIMAL(18,2)` — 2 decimal places, large total range |
| Should a default be provided? | Revenue/count columns default to 0; CreatedDate defaults to today |

These decisions are not arbitrary. Choosing `INT` instead of `TINYINT` for `MonthNumber` wastes 3 bytes per row — small at 100 rows, noticeable at 100 million. Choosing `FLOAT` instead of `DECIMAL` for revenue invites rounding errors. Taking 10 minutes to think through these choices before writing the DDL is time well spent.

---

## 3. Constraints: Enforcing Business Rules

**Constraints** are rules applied to a table that the database engine enforces automatically. Every row inserted or updated must satisfy all constraints — violations are rejected with an error before any data is changed.

### 3.1 Primary Key Constraint

The primary key uniquely identifies each row. No two rows can have the same primary key value, and the column cannot be NULL:

```sql
CREATE TABLE dbo.MonthlySalesSummary
(
    SummaryID       INT             NOT NULL IDENTITY(1,1),
    CalendarYear    SMALLINT        NOT NULL,
    ...
    CONSTRAINT PK_MonthlySalesSummary
        PRIMARY KEY (SummaryID)
);
```

Naming constraints explicitly (`PK_MonthlySalesSummary`) makes them identifiable in error messages and system catalog queries. SQL Server generates names for unnamed constraints, but they are cryptic and hard to reference.

### 3.2 Unique Constraint

A unique constraint ensures no two rows have the same value(s) in the specified column(s):

```sql
-- No two rows can have the same year + month + territory combination
CONSTRAINT UQ_MonthlySalesSummary_Period
    UNIQUE (CalendarYear, MonthNumber, SalesTerritory)
```

This prevents duplicate summary rows — a critical data quality rule for summary tables. Attempting to insert a second row for 'Atlantic — Nova Scotia', year 2024, month 3 will fail.

### 3.3 CHECK Constraint

A check constraint validates that column values satisfy a logical condition:

```sql
-- Month must be between 1 and 12
CONSTRAINT CK_MonthlySalesSummary_Month
    CHECK (MonthNumber BETWEEN 1 AND 12),

-- Revenue cannot be negative
CONSTRAINT CK_MonthlySalesSummary_Revenue
    CHECK (TotalRevenue >= 0),

-- Margin percentage must be between 0 and 100 (or NULL)
CONSTRAINT CK_MonthlySalesSummary_Margin
    CHECK (AvgMarginPct IS NULL OR AvgMarginPct BETWEEN 0 AND 100)
```

### 3.4 Foreign Key Constraint

A foreign key links a column in one table to the primary key of another, enforcing referential integrity:

```sql
-- If SummaryID references another table's FK, a foreign key constraint enforces the link
CONSTRAINT FK_SomeTable_Summary
    FOREIGN KEY (SummaryID) REFERENCES dbo.MonthlySalesSummary (SummaryID)
```

For standalone summary tables like `MonthlySalesSummary`, foreign keys to the source dimension tables are optional — they are useful for enforcing that `CalendarYear` and `MonthNumber` values are valid, but they create a coupling between the summary table and the dimension tables that may not always be desirable.

### 3.5 The Complete CREATE TABLE with All Constraints

```sql
DROP TABLE IF EXISTS dbo.MonthlySalesSummary;
GO

CREATE TABLE dbo.MonthlySalesSummary
(
    SummaryID           INT             NOT NULL IDENTITY(1,1),
    CalendarYear        SMALLINT        NOT NULL,
    MonthNumber         TINYINT         NOT NULL,
    MonthName           NVARCHAR(10)    NOT NULL,
    SalesTerritory      NVARCHAR(60)    NOT NULL,
    InvoiceCount        INT             NOT NULL    DEFAULT 0,
    SalesLineCount      INT             NOT NULL    DEFAULT 0,
    TotalRevenue        DECIMAL(18,2)   NOT NULL    DEFAULT 0,
    TotalGrossProfit    DECIMAL(18,2)   NOT NULL    DEFAULT 0,
    AvgMarginPct        DECIMAL(8,2)    NULL,
    CreatedDate         DATE            NOT NULL    DEFAULT GETDATE(),

    CONSTRAINT PK_MonthlySalesSummary
        PRIMARY KEY (SummaryID),
    CONSTRAINT UQ_MonthlySalesSummary_Period
        UNIQUE (CalendarYear, MonthNumber, SalesTerritory),
    CONSTRAINT CK_MonthlySalesSummary_Month
        CHECK (MonthNumber BETWEEN 1 AND 12),
    CONSTRAINT CK_MonthlySalesSummary_Revenue
        CHECK (TotalRevenue >= 0),
    CONSTRAINT CK_MonthlySalesSummary_Margin
        CHECK (AvgMarginPct IS NULL OR AvgMarginPct BETWEEN 0 AND 100)
);
GO
```

The `DROP TABLE IF EXISTS` at the top makes the script safe to re-run — if the table already exists, it is dropped first. Without this, re-running the script would fail with "table already exists."

---

## 4. ALTER TABLE: Modifying Structures

`ALTER TABLE` modifies an existing table without rebuilding it from scratch.

### 4.1 Adding a Column

```sql
-- Add a column after the table exists
ALTER TABLE dbo.MonthlySalesSummary
ADD AvgInvoiceValue DECIMAL(18,2) NULL;
-- New columns must be NULL or have a DEFAULT when the table already has rows
-- (otherwise the database cannot populate the column for existing rows)
```

### 4.2 Changing a Column's Data Type

```sql
-- Increase precision of an existing column
ALTER TABLE dbo.MonthlySalesSummary
ALTER COLUMN AvgMarginPct DECIMAL(10,4) NULL;
-- Changing to higher precision is safe
-- Changing to lower precision may truncate existing data
```

### 4.3 Adding and Dropping Constraints

```sql
-- Add a constraint after creation
ALTER TABLE dbo.MonthlySalesSummary
ADD CONSTRAINT CK_MonthlySalesSummary_Year
    CHECK (CalendarYear BETWEEN 2000 AND 2100);

-- Drop a constraint by name
ALTER TABLE dbo.MonthlySalesSummary
DROP CONSTRAINT CK_MonthlySalesSummary_Year;
```

### 4.4 Renaming a Column

SQL Server uses a system procedure to rename columns:

```sql
-- Rename a column
EXEC sp_rename
    'dbo.MonthlySalesSummary.AvgInvoiceValue',
    'AvgLineValue',
    'COLUMN';
```

> **Renaming carries risk:** any query, view, or stored procedure that references the old column name will break immediately. Always search for references before renaming a column in a production database.

---

## 5. DROP TABLE: Removing Structures

`DROP TABLE` permanently removes the table and all its data:

```sql
-- Safe version: only drops if the table exists
DROP TABLE IF EXISTS dbo.MonthlySalesSummary;

-- Unsafe version: raises an error if the table does not exist
DROP TABLE dbo.MonthlySalesSummary;
```

**DROP TABLE is permanent.** There is no undo. Always verify you are dropping the right table in the right database before executing.

`TRUNCATE TABLE` removes all rows but keeps the table structure:

```sql
-- Remove all data, keep the table
TRUNCATE TABLE dbo.MonthlySalesSummary;

-- Compare: DROP removes both data and structure
-- TRUNCATE removes data only
```

`TRUNCATE` is faster than `DELETE` for clearing large tables — it deallocates data pages without logging individual row deletions. It cannot be used when FK constraints reference the table from other tables with data.

---

## 6. DML Revisited: INSERT, UPDATE, DELETE

Chapter 3 introduced DML concepts. This section extends them with the patterns you need for source-to-target work.

### 6.1 INSERT with Multiple Rows

`INSERT INTO ... VALUES` can insert multiple rows in a single statement:

```sql
INSERT INTO dbo.MonthlySalesSummary
    (CalendarYear, MonthNumber, MonthName, SalesTerritory,
     InvoiceCount, SalesLineCount, TotalRevenue, TotalGrossProfit, AvgMarginPct)
VALUES
    (2024, 1, 'January', 'Atlantic — Nova Scotia', 12, 87, 45230.50, 14925.07, 33.00),
    (2024, 1, 'January', 'Ontario',                 8, 54, 29100.00, 9603.00,  33.00),
    (2024, 2, 'February', 'Atlantic — Nova Scotia', 15, 103, 51420.75, 16968.85, 33.00);
-- Three rows inserted in one statement
```

### 6.2 UPDATE with the SELECT-First Discipline

Always verify the target rows with a `SELECT` before running `UPDATE`:

```sql
-- Step 1: See what you are about to update
SELECT  SummaryID, TotalRevenue, AvgMarginPct
FROM    dbo.MonthlySalesSummary
WHERE   CalendarYear  = 2024
AND     MonthNumber   = 1
AND     SalesTerritory = 'Atlantic — Nova Scotia';

-- Step 2: Run the UPDATE with the identical WHERE clause
UPDATE  dbo.MonthlySalesSummary
SET     AvgMarginPct = 32.95
WHERE   CalendarYear  = 2024
AND     MonthNumber   = 1
AND     SalesTerritory = 'Atlantic — Nova Scotia';

-- Step 3: Verify the change
SELECT  SummaryID, TotalRevenue, AvgMarginPct
FROM    dbo.MonthlySalesSummary
WHERE   CalendarYear  = 2024
AND     MonthNumber   = 1
AND     SalesTerritory = 'Atlantic — Nova Scotia';
```

### 6.3 UPDATE from a JOIN

`UPDATE` can reference another table using a JOIN — useful for updating a target table from source data:

```sql
-- Update TotalRevenue in the summary table from actual fact table data
UPDATE  mss
SET     mss.TotalRevenue = src.Revenue
FROM    dbo.MonthlySalesSummary mss
INNER JOIN (
    SELECT  cal.CalendarYear,
            cal.MonthNumber,
            cust.SalesTerritory,
            SUM(fs.LineTotal)   AS Revenue
    FROM    fact.Sales fs
    INNER JOIN dim.Calendar  cal  ON cal.DateKey      = fs.OrderDateKey
    INNER JOIN dim.Customer  cust ON cust.CustomerKey = fs.CustomerKey
    GROUP BY cal.CalendarYear, cal.MonthNumber, cust.SalesTerritory
) src
    ON  src.CalendarYear   = mss.CalendarYear
    AND src.MonthNumber    = mss.MonthNumber
    AND src.SalesTerritory = mss.SalesTerritory;
```

This pattern — `UPDATE target FROM target JOIN (source query)` — is the T-SQL equivalent of an SSIS `OLE DB Command` transformation. It updates the target table in place rather than rebuilding it from scratch.

---

## 7. INSERT INTO … SELECT: Populating from a Query

`INSERT INTO ... SELECT` is the most important DML pattern for analytical SQL. It populates a table from any SELECT query — the entire ETL source-to-target process in a single statement.

### 7.1 Syntax

```sql
INSERT INTO target_table (col1, col2, col3, ...)
SELECT  expression1, expression2, expression3, ...
FROM    source_table
WHERE   condition;
```

The columns in the INSERT list and the expressions in the SELECT list must match in number and compatible data types. The SELECT can be any valid query — including joins, aggregations, subqueries, CTEs, and window functions.

### 7.2 Populating MonthlySalesSummary

```sql
-- Clear the table first (safe to re-run pattern)
TRUNCATE TABLE dbo.MonthlySalesSummary;

-- Populate from the star schema
INSERT INTO dbo.MonthlySalesSummary
(
    CalendarYear, MonthNumber, MonthName, SalesTerritory,
    InvoiceCount, SalesLineCount, TotalRevenue, TotalGrossProfit, AvgMarginPct
)
SELECT
    cal.CalendarYear,
    cal.MonthNumber,
    cal.MonthName,
    cust.SalesTerritory,
    COUNT(DISTINCT fs.InvoiceID)                                AS InvoiceCount,
    COUNT(*)                                                    AS SalesLineCount,
    SUM(fs.LineTotal)                                           AS TotalRevenue,
    SUM(fs.GrossProfit)                                         AS TotalGrossProfit,
    ROUND(
        SUM(fs.GrossProfit) / NULLIF(SUM(fs.LineTotal), 0) * 100,
    2)                                                          AS AvgMarginPct
FROM    fact.Sales fs
INNER JOIN dim.Calendar  cal  ON cal.DateKey      = fs.OrderDateKey
INNER JOIN dim.Customer  cust ON cust.CustomerKey = fs.CustomerKey
GROUP BY
    cal.CalendarYear,
    cal.MonthNumber,
    cal.MonthName,
    cust.SalesTerritory;
```

**Note:** `SummaryID` and `CreatedDate` are not in the INSERT column list — they have `IDENTITY` and `DEFAULT GETDATE()` respectively, so SQL Server populates them automatically.

### 7.3 Verifying the Insert

```sql
-- Check row count
SELECT COUNT(*) AS RowsInserted FROM dbo.MonthlySalesSummary;

-- Sample the results
SELECT TOP 10 *
FROM   dbo.MonthlySalesSummary
ORDER BY CalendarYear, MonthNumber, SalesTerritory;

-- Cross-check: total revenue in summary should match total revenue in fact table
SELECT
    'Source (fact.Sales)'               AS Layer,
    ROUND(SUM(LineTotal), 2)            AS TotalRevenue
FROM    fact.Sales
UNION ALL
SELECT
    'Target (MonthlySalesSummary)',
    ROUND(SUM(TotalRevenue), 2)
FROM    dbo.MonthlySalesSummary;
-- Both values must be equal
```

The reconciliation query — comparing source and target aggregate values — is the fundamental quality check for any source-to-target operation. If the totals match, the data moved correctly. If they differ, there is a defect to find.

---

## 8. SELECT INTO: Rapid Prototyping

`SELECT INTO` creates a new table and inserts rows in a single statement — faster to write than the two-step `CREATE TABLE` + `INSERT INTO ... SELECT`:

```sql
-- Create and populate a new table in one step
SELECT
    cal.CalendarYear,
    cal.MonthNumber,
    cal.MonthName,
    cust.SalesTerritory,
    COUNT(DISTINCT fs.InvoiceID)    AS InvoiceCount,
    SUM(fs.LineTotal)               AS TotalRevenue
INTO    dbo.MonthlySalesSummaryQuick    -- New table created here
FROM    fact.Sales fs
INNER JOIN dim.Calendar  cal  ON cal.DateKey      = fs.OrderDateKey
INNER JOIN dim.Customer  cust ON cust.CustomerKey = fs.CustomerKey
GROUP BY cal.CalendarYear, cal.MonthNumber, cal.MonthName, cust.SalesTerritory;

-- Verify
SELECT TOP 5 * FROM dbo.MonthlySalesSummaryQuick ORDER BY CalendarYear, MonthNumber;
```

### 8.1 What SELECT INTO Does Not Create

`SELECT INTO` creates the table with:
- The column names and data types derived from the SELECT list
- No primary key
- No indexes
- No constraints
- No defaults (except those inherited from the source if using `IDENTITY` columns)

For prototyping and one-off analysis, this is fine. For production tables that must enforce data integrity and perform well under query load, always use `CREATE TABLE` with explicit constraints followed by `INSERT INTO ... SELECT`.

### 8.2 SELECT INTO a Temporary Table

`SELECT INTO` with a `#` prefix creates a **temporary table** in `tempdb`:

```sql
-- Create a temporary table for intermediate calculations
SELECT
    fs.CustomerKey,
    SUM(fs.LineTotal)   AS TotalRevenue
INTO    #CustomerTotals     -- # prefix = temporary table in tempdb
FROM    fact.Sales fs
GROUP BY fs.CustomerKey;

-- Use the temp table in subsequent queries
SELECT  cust.CustomerName,
        ct.TotalRevenue
FROM    #CustomerTotals ct
INNER JOIN dim.Customer cust ON cust.CustomerKey = ct.CustomerKey
ORDER BY ct.TotalRevenue DESC;

-- Clean up
DROP TABLE IF EXISTS #CustomerTotals;
```

Temporary tables exist only for the current session and are automatically dropped when the session ends. They are ideal for breaking a complex multi-step query into manageable pieces when CTEs are not sufficient — for example, when the intermediate result needs to be referenced by multiple subsequent queries or benefits from an index.

---

## 9. Writing a Source-to-Target Script

A **source-to-target script** is a complete, self-contained SQL script that:
1. Creates the target structure (DDL)
2. Clears any existing data
3. Populates the target from the source
4. Verifies the result

This is the T-SQL equivalent of what SSIS packages do — the same logical steps, expressed as SQL statements rather than graphical components.

### 9.1 A Complete Source-to-Target Script

```sql
-- ============================================================
-- Script: Monthly Sales Rep Performance Summary
-- Source:  CabotTrailOutdoorsSales (fact.Sales + dimensions)
-- Target:  dbo.SalesRepMonthlySummary
-- Author:  [Your Name]
-- Date:    [Date]
-- Purpose: Creates and populates a monthly performance table
--          for use in sales rep dashboards
-- ============================================================

USE CabotTrailOutdoorsSales;
GO

-- ── Step 1: Create the target table ────────────────────────
DROP TABLE IF EXISTS dbo.SalesRepMonthlySummary;
GO

CREATE TABLE dbo.SalesRepMonthlySummary
(
    SummaryKey          INT             NOT NULL IDENTITY(1,1),
    SalesRepName        NVARCHAR(100)   NOT NULL,
    Department          NVARCHAR(60)    NULL,
    CalendarYear        SMALLINT        NOT NULL,
    MonthNumber         TINYINT         NOT NULL,
    MonthName           NVARCHAR(10)    NOT NULL,
    YearMonthLabel      NVARCHAR(15)    NOT NULL,
    InvoiceCount        INT             NOT NULL    DEFAULT 0,
    SalesLineCount      INT             NOT NULL    DEFAULT 0,
    TotalRevenue        DECIMAL(18,2)   NOT NULL    DEFAULT 0,
    TotalGrossProfit    DECIMAL(18,2)   NOT NULL    DEFAULT 0,
    AvgMarginPct        DECIMAL(8,2)    NULL,
    AvgDaysToInvoice    DECIMAL(6,2)    NULL,

    CONSTRAINT PK_SalesRepMonthlySummary
        PRIMARY KEY (SummaryKey),
    CONSTRAINT UQ_SalesRepMonthlySummary_Period
        UNIQUE (SalesRepName, CalendarYear, MonthNumber),
    CONSTRAINT CK_SalesRepMonthlySummary_Month
        CHECK (MonthNumber BETWEEN 1 AND 12),
    CONSTRAINT CK_SalesRepMonthlySummary_Revenue
        CHECK (TotalRevenue >= 0)
);
GO

-- ── Step 2: Populate from source ───────────────────────────
INSERT INTO dbo.SalesRepMonthlySummary
(
    SalesRepName, Department,
    CalendarYear, MonthNumber, MonthName, YearMonthLabel,
    InvoiceCount, SalesLineCount,
    TotalRevenue, TotalGrossProfit, AvgMarginPct, AvgDaysToInvoice
)
SELECT
    emp.FullName                                AS SalesRepName,
    emp.Department,
    cal.CalendarYear,
    cal.MonthNumber,
    cal.MonthName,
    -- Build a readable period label: 'Jan 2024'
    LEFT(cal.MonthName, 3) + ' '
        + CAST(cal.CalendarYear AS NVARCHAR(4))  AS YearMonthLabel,
    COUNT(DISTINCT fs.InvoiceID)                AS InvoiceCount,
    COUNT(*)                                    AS SalesLineCount,
    SUM(fs.LineTotal)                           AS TotalRevenue,
    SUM(fs.GrossProfit)                         AS TotalGrossProfit,
    ROUND(
        SUM(fs.GrossProfit) / NULLIF(SUM(fs.LineTotal), 0) * 100,
    2)                                          AS AvgMarginPct,
    ROUND(AVG(CAST(fs.DaysUntilInvoice AS DECIMAL(6,2))), 2)
                                                AS AvgDaysToInvoice
FROM    fact.Sales fs
INNER JOIN dim.Employee emp  ON emp.EmployeeKey  = fs.EmployeeKey
INNER JOIN dim.Calendar  cal ON cal.DateKey      = fs.OrderDateKey
GROUP BY
    emp.FullName, emp.Department,
    cal.CalendarYear, cal.MonthNumber, cal.MonthName;
GO

-- ── Step 3: Verify ─────────────────────────────────────────

-- 3a: Row count
SELECT  COUNT(*)    AS RowsLoaded FROM dbo.SalesRepMonthlySummary;

-- 3b: Revenue reconciliation (source vs target totals must match)
SELECT
    'Source (fact.Sales)'           AS Layer,
    ROUND(SUM(LineTotal), 2)        AS TotalRevenue,
    COUNT(DISTINCT InvoiceID)       AS TotalInvoices
FROM    fact.Sales
UNION ALL
SELECT
    'Target (SalesRepMonthlySummary)',
    ROUND(SUM(TotalRevenue), 2),
    SUM(InvoiceCount)
FROM    dbo.SalesRepMonthlySummary;

-- 3c: Sample output
SELECT TOP 15 *
FROM   dbo.SalesRepMonthlySummary
ORDER BY CalendarYear, MonthNumber, SalesRepName;
GO
```

### 9.2 The Anatomy of a Good Source-to-Target Script

Every production-quality source-to-target script has the same anatomy:

| Section | Purpose |
|---|---|
| **Comment block** | Documents purpose, source, target, author, date |
| `USE` statement | Ensures the right database is active |
| **Step 1: DDL** | `DROP IF EXISTS` + `CREATE TABLE` with all constraints named |
| **Step 2: Load** | `TRUNCATE` or `DELETE`, then `INSERT INTO ... SELECT` |
| **Step 3: Verify** | Row counts, aggregate reconciliation, sample rows |

The `DROP IF EXISTS` + `CREATE TABLE` pattern makes scripts **idempotent** — safe to run multiple times without error. Each run starts fresh.

---

## 10. Verifying Your Work: Reconciliation Queries

Reconciliation is the practice of verifying that data moved correctly from source to target. Three levels of verification, in increasing depth:

### 10.1 Level 1: Row Count

```sql
-- Does the target have the expected number of rows?
SELECT COUNT(*) AS TargetRows FROM dbo.SalesRepMonthlySummary;

-- Compare to source: how many (SalesRep, Year, Month) combinations exist?
SELECT COUNT(*) AS SourceCombinations
FROM (
    SELECT DISTINCT
        fs.EmployeeKey,
        cal.CalendarYear,
        cal.MonthNumber
    FROM    fact.Sales fs
    INNER JOIN dim.Calendar cal ON cal.DateKey = fs.OrderDateKey
) AS Combinations;
-- TargetRows should equal SourceCombinations
```

### 10.2 Level 2: Aggregate Reconciliation

```sql
-- Does total revenue in the target match total revenue in the source?
SELECT
    'Source' AS Layer, ROUND(SUM(LineTotal), 2) AS Revenue
FROM    fact.Sales
UNION ALL
SELECT
    'Target', ROUND(SUM(TotalRevenue), 2)
FROM    dbo.SalesRepMonthlySummary;
-- Both must return the same value
```

### 10.3 Level 3: Spot Check

```sql
-- Pick a specific (SalesRep, Year, Month) and verify each measure independently
DECLARE @Rep   NVARCHAR(100) = 'Alice MacLean';
DECLARE @Year  SMALLINT      = 2024;
DECLARE @Month TINYINT       = 3;

-- What the source says
SELECT
    COUNT(DISTINCT InvoiceID)               AS SourceInvoices,
    COUNT(*)                                AS SourceLines,
    ROUND(SUM(LineTotal), 2)                AS SourceRevenue,
    ROUND(SUM(GrossProfit)
          / NULLIF(SUM(LineTotal),0)*100,2) AS SourceMargin
FROM    fact.Sales fs
INNER JOIN dim.Employee emp ON emp.EmployeeKey = fs.EmployeeKey
INNER JOIN dim.Calendar  cal ON cal.DateKey     = fs.OrderDateKey
WHERE   emp.FullName      = @Rep
AND     cal.CalendarYear  = @Year
AND     cal.MonthNumber   = @Month;

-- What the target says
SELECT  InvoiceCount, SalesLineCount, TotalRevenue, AvgMarginPct
FROM    dbo.SalesRepMonthlySummary
WHERE   SalesRepName  = @Rep
AND     CalendarYear  = @Year
AND     MonthNumber   = @Month;
```

If all three levels pass — row counts match, aggregates match, spot checks agree — the source-to-target script is verified correct.

---

## 11. From T-SQL Scripts to SSIS Packages

Everything in this chapter has been implemented as T-SQL. In DBAS 2103 (Data Provisioning with ETL), you will build the same pipelines as SSIS packages. Understanding the relationship between the two approaches is the conceptual bridge between this course and the next.

### 11.1 The Same Steps, Different Tools

| T-SQL step | SSIS equivalent |
|---|---|
| `USE DatabaseName` | OLE DB Connection Manager pointing to the database |
| `TRUNCATE TABLE target` | Execute SQL Task in Control Flow |
| `INSERT INTO ... SELECT` | Data Flow Task: OLE DB Source → OLE DB Destination |
| Complex joins in SELECT | Multiple tables in OLE DB Source SQL or Lookup transformations |
| `NULLIF` for divide-by-zero | Derived Column transformation with SSIS expression |
| Verification `SELECT COUNT(*)` | Execute SQL Task with result stored in variable |
| Mismatch alert | SSIS Failure path + Send Mail Task |

The SSIS package adds **monitoring** (every execution is logged in the SSIS Catalog), **error handling** (individual rows can be redirected to error outputs), and **scheduling** (SQL Server Agent runs the package automatically). The T-SQL script adds none of these — it runs once when you press F5.

### 11.2 Why Learn Both

T-SQL source-to-target scripts are:
- Faster to write for ad-hoc or one-time loads
- Easier to understand, debug, and modify
- Ideal for prototyping before building the SSIS equivalent
- Version-controllable as plain text files
- Runnable by anyone with SSMS access

SSIS packages are:
- More appropriate for scheduled production loads
- Better for large-volume loads (streaming architecture)
- Superior for error handling at the row level
- Integrated with the monitoring and alerting infrastructure

The T-SQL script from section 9.1 and an SSIS package doing the same work are functionally equivalent. Many ETL pipelines in production use both — T-SQL for simple lookups and small loads, SSIS for the main fact table loads.

### 11.3 The Source-to-Target Mapping Connects Both

Whether you implement a data movement in T-SQL or SSIS, the specification comes from the same place: the **source-to-target mapping** document introduced in DBAS 2010. The mapping defines what each target column contains and how it is derived from the source. The T-SQL SELECT clause implements the mapping in SQL; the SSIS Data Flow implements the same mapping visually. The mapping is the constant; the implementation tool varies.

---

## 12. Looking Back, Looking Forward

### 12.1 What This Course Covered

Over nine chapters, this book has taken you from "what is a database?" to writing complete source-to-target pipelines. The journey:

| Chapter | Core skill |
|---|---|
| 1 | Connecting to a database and writing SELECT |
| 2 | Filtering rows precisely with WHERE |
| 3 | Data types, arithmetic, and introductory DML |
| 4 | String, date, and NULL functions; CASE expressions |
| 5 | Joining tables; the star schema pattern |
| 6 | Aggregation: GROUP BY, HAVING, aggregate functions |
| 7 | Scripting: variables, subqueries, CTEs |
| 8 | Window functions, UDFs, and stored procedures |
| 9 | DDL, DML, and source-to-target scripts |

### 12.2 The Analytical Foundation

The DQL skills — from basic SELECT through window functions — are the tools of the practising data analyst. They answer questions, support decisions, and surface insights from data that would otherwise remain invisible in rows and columns.

The DDL and DML skills — from CREATE TABLE through INSERT INTO ... SELECT — are the tools of the data engineer and the analyst who needs to move and transform data. They are the foundation of ETL work.

Together, they constitute the SQL literacy that makes a data professional effective in any environment with a relational database.

### 12.3 Where to Go Next

The three-course sequence continues:

**DBAS 2010 — Database Design II** applies the data type and constraint knowledge from this course to designing relational databases from business requirements. You will move from *querying* the CabotTrail databases to *designing* databases like them — entity-relationship diagrams, normalization, physical implementation.

**DBAS 2103 — Data Provisioning with ETL** builds the SSIS pipelines that implement the source-to-target patterns introduced in this chapter at production scale — with proper error handling, scheduling, monitoring, and governance.

The companion textbook *ETL for Business Intelligence* (the same author, same CabotTrail environment) is the reference for DBAS 2103 and covers all of these topics in depth.

---

## 13. Chapter Summary

- **DDL** creates and modifies database structures: `CREATE TABLE`, `ALTER TABLE`, `DROP TABLE`, `TRUNCATE TABLE`.

- `CREATE TABLE` defines column names, data types, nullability, and defaults. Choose the smallest data type that can represent all valid values; choose `DECIMAL` for financial amounts; use `NVARCHAR` for text.

- **Constraints** enforce business rules: `PRIMARY KEY` (unique, not null identifier), `UNIQUE` (no duplicate combinations), `CHECK` (value must satisfy a condition), `FOREIGN KEY` (references another table's PK). Always name constraints explicitly.

- `DROP TABLE IF EXISTS` before `CREATE TABLE` makes scripts idempotent — safe to re-run.

- **`INSERT INTO ... SELECT`** is the core source-to-target pattern: defines target columns, queries the source, and populates the target in one statement. Always `TRUNCATE` or `DELETE` before re-populating.

- **`SELECT INTO`** creates a table and populates it in one step — convenient for prototyping. It does not create indexes, constraints, or primary keys.

- A complete **source-to-target script** has four sections: comment block, DDL (create target), DML (populate from source), verification (row counts and aggregate reconciliation).

- **Reconciliation** verifies the load at three levels: row count, aggregate total, and spot check. All three must pass before the load is considered correct.

- T-SQL source-to-target scripts and SSIS packages implement the same logical steps with different tools. T-SQL is faster for prototyping; SSIS adds monitoring, error handling, and scheduling for production.

---

## 14. Review Questions

1. Write a `CREATE TABLE` statement for a table called `dbo.ProductPerformanceSummary` that stores total revenue, total units sold, and return rate by product and year. Include appropriate data types, a primary key, a unique constraint preventing duplicate (product, year) rows, and at least two CHECK constraints. Name all constraints explicitly.

2. Explain the difference between `DROP TABLE`, `TRUNCATE TABLE`, and `DELETE FROM table_name` (with no WHERE clause). When would you use each, and what are the key distinctions in terms of speed, recoverability, and FK constraint behaviour?

3. Write a complete source-to-target script (all four sections: comment block, DDL, populate, verify) that creates and populates `dbo.TerritoryQuarterlySummary` from `CabotTrailOutdoorsSales`. The target should have one row per (SalesTerritory, CalendarYear, QuarterNumber) with total revenue, invoice count, and average margin. Include the full reconciliation check.

4. A developer uses `SELECT INTO` to create a reporting table and then complains that queries against it are slow. Identify two specific things `SELECT INTO` does not create that would help query performance, and show the SQL to add them after the fact.

5. Write an `UPDATE` statement that corrects `AvgMarginPct` in `dbo.SalesRepMonthlySummary` by recomputing it from `fact.Sales`. Use the SELECT-first discipline — show the SELECT step that identifies the rows to be updated before running the UPDATE.

6. A reconciliation check shows that the target table has 50 rows but the source query (counting distinct SalesRep + Year + Month combinations) returns 52. What are the two most likely causes of this discrepancy, and how would you diagnose each?

7. Explain why `INSERT INTO dbo.SalesRepMonthlySummary (...) SELECT ...` requires the column list `(...)` to be specified explicitly, even though the SELECT list has the same columns in the same order. When is the column list safe to omit, and when is it risky?

8. The companion textbook *ETL for Business Intelligence* describes a T-SQL source-to-target script and an SSIS package as "functionally equivalent." Describe two specific scenarios where the SSIS approach is clearly preferable to a T-SQL script, and one scenario where the T-SQL script is preferable.

---

## 🔍 Deeper Dive

### Going Further with DDL, DML, and Data Movement

#### Schema Design Principles

This chapter introduced enough DDL to build functional tables. Designing *good* tables — ones that support the queries you need, perform well, and remain maintainable — requires applying the normalization principles covered in DBAS 2010.

For summary tables like `SalesRepMonthlySummary`, the key design questions are:

**Grain:** What does one row represent? (One sales rep + one month = one row.) The grain determines the primary key candidate.

**Measures:** Are all stored values truly additive? (`TotalRevenue` is additive; `AvgMarginPct` is not — it should be derived at query time from the additive components, or at minimum documented as non-additive.)

**Freshness:** How often is the table repopulated? Does a TRUNCATE + full reload make sense, or is an incremental MERGE approach needed?

These questions connect directly to the dimensional modelling concepts in the ETL book's Chapters 2 and 3.

#### MERGE: The Upsert Pattern

Chapter 7 of *ETL for Business Intelligence* covered the `MERGE` statement in depth. For source-to-target scripts that need to **update existing rows and insert new ones** rather than truncate and reload, `MERGE` is the standard tool:

```sql
-- MERGE: update existing rows, insert new ones
MERGE INTO dbo.SalesRepMonthlySummary AS tgt
USING (
    SELECT
        emp.FullName                            AS SalesRepName,
        emp.Department,
        cal.CalendarYear, cal.MonthNumber, cal.MonthName,
        LEFT(cal.MonthName,3) + ' ' + CAST(cal.CalendarYear AS NVARCHAR(4)) AS YearMonthLabel,
        COUNT(DISTINCT fs.InvoiceID)            AS InvoiceCount,
        COUNT(*)                                AS SalesLineCount,
        SUM(fs.LineTotal)                       AS TotalRevenue,
        SUM(fs.GrossProfit)                     AS TotalGrossProfit,
        ROUND(SUM(fs.GrossProfit)
              / NULLIF(SUM(fs.LineTotal),0)*100, 2) AS AvgMarginPct,
        ROUND(AVG(CAST(fs.DaysUntilInvoice AS DECIMAL(6,2))),2) AS AvgDaysToInvoice
    FROM    fact.Sales fs
    INNER JOIN dim.Employee emp ON emp.EmployeeKey = fs.EmployeeKey
    INNER JOIN dim.Calendar  cal ON cal.DateKey    = fs.OrderDateKey
    GROUP BY emp.FullName, emp.Department,
             cal.CalendarYear, cal.MonthNumber, cal.MonthName
) AS src
ON (    tgt.SalesRepName  = src.SalesRepName
    AND tgt.CalendarYear  = src.CalendarYear
    AND tgt.MonthNumber   = src.MonthNumber)
-- Row exists: update if measures have changed
WHEN MATCHED AND tgt.TotalRevenue <> src.TotalRevenue THEN
    UPDATE SET
        tgt.InvoiceCount     = src.InvoiceCount,
        tgt.SalesLineCount   = src.SalesLineCount,
        tgt.TotalRevenue     = src.TotalRevenue,
        tgt.TotalGrossProfit = src.TotalGrossProfit,
        tgt.AvgMarginPct     = src.AvgMarginPct
-- Row does not exist: insert new
WHEN NOT MATCHED BY TARGET THEN
    INSERT (SalesRepName, Department, CalendarYear, MonthNumber, MonthName,
            YearMonthLabel, InvoiceCount, SalesLineCount,
            TotalRevenue, TotalGrossProfit, AvgMarginPct, AvgDaysToInvoice)
    VALUES (src.SalesRepName, src.Department, src.CalendarYear, src.MonthNumber,
            src.MonthName, src.YearMonthLabel, src.InvoiceCount, src.SalesLineCount,
            src.TotalRevenue, src.TotalGrossProfit, src.AvgMarginPct, src.AvgDaysToInvoice);
```

`MERGE` is the bridge between the simple TRUNCATE + INSERT approach (appropriate for small, fast-loading tables) and the incremental update approach (appropriate for large tables or when preserving history matters). You will use it extensively in DBAS 2103.

Microsoft documentation:
[MERGE (Transact-SQL)](https://learn.microsoft.com/en-us/sql/t-sql/statements/merge-transact-sql)

#### Transactions for Multi-Step Scripts

When a source-to-target script involves multiple DML statements that must all succeed or all fail together, wrap them in a transaction:

```sql
BEGIN TRANSACTION;

    -- Step 1: Clear target
    TRUNCATE TABLE dbo.SalesRepMonthlySummary;

    -- Step 2: Populate
    INSERT INTO dbo.SalesRepMonthlySummary (...) SELECT ...;

    -- Step 3: Verify before committing
    DECLARE @SourceRev DECIMAL(18,2);
    DECLARE @TargetRev DECIMAL(18,2);

    SELECT @SourceRev = ROUND(SUM(LineTotal), 2) FROM fact.Sales;
    SELECT @TargetRev = ROUND(SUM(TotalRevenue), 2) FROM dbo.SalesRepMonthlySummary;

    IF @SourceRev = @TargetRev
    BEGIN
        PRINT 'Reconciliation passed. Committing.';
        COMMIT;
    END
    ELSE
    BEGIN
        PRINT 'Reconciliation FAILED. Rolling back. Source: '
              + CAST(@SourceRev AS NVARCHAR)
              + ' Target: ' + CAST(@TargetRev AS NVARCHAR);
        ROLLBACK;
    END
```

This pattern makes the entire load atomic — the target is either fully updated with verified data, or exactly as it was before (the truncate is also rolled back). It is the T-SQL equivalent of SSIS's transactional package execution.

Microsoft documentation:
[Transactions (Transact-SQL)](https://learn.microsoft.com/en-us/sql/t-sql/language-elements/transactions-transact-sql)

#### Indexes on Summary Tables

`CREATE TABLE` creates the structure; `CREATE INDEX` makes it fast. For a summary table queried by analysts and BI tools:

```sql
-- Primary key creates a clustered index automatically (data sorted by SummaryKey)
-- Add non-clustered indexes for common filter/join columns

CREATE NONCLUSTERED INDEX IX_SalesRepMonthlySummary_Rep
    ON dbo.SalesRepMonthlySummary (SalesRepName);

CREATE NONCLUSTERED INDEX IX_SalesRepMonthlySummary_Period
    ON dbo.SalesRepMonthlySummary (CalendarYear, MonthNumber);

-- Covering index: includes measure columns for a common query pattern
CREATE NONCLUSTERED INDEX IX_SalesRepMonthlySummary_Rep_Period_Rev
    ON dbo.SalesRepMonthlySummary (SalesRepName, CalendarYear, MonthNumber)
    INCLUDE (TotalRevenue, InvoiceCount);
```

The last index is a **covering index** — it includes the measure columns that a common query needs, so SQL Server can answer the query entirely from the index without reading the main table data. For analytical queries that consistently access the same set of columns, covering indexes dramatically improve performance.

---

### Industry Perspectives

#### Source-to-Target Scripts in the Real World

In production data environments, T-SQL source-to-target scripts serve three common purposes:

**Prototyping:** A data engineer writes a T-SQL script to verify logic before investing in a full SSIS package. The script proves the joins are correct, the aggregations produce the right numbers, and the target schema supports the required queries. Only then is the SSIS package built.

**One-time loads:** Historical data migrations, backfills, and one-off reporting tables that run once and never need scheduling. A T-SQL script is appropriate when the operational overhead of maintaining an SSIS package is not justified.

**Data quality corrections:** When a production ETL has loaded incorrect data, a T-SQL script may be used to correct it in-place — an UPDATE from a corrected source — rather than rerunning the full ETL.

Understanding when T-SQL is the right tool and when SSIS is the right tool is a professional judgment that comes with experience. This chapter gives you the T-SQL skills to make that judgment informed.

---

### References and Further Reading

1. Ben-Gan, I. (2015). *T-SQL Fundamentals* (3rd ed.). Microsoft Press. — Chapters 10–11 cover DDL and DML in depth, including constraints, identity columns, and sequences.

2. Microsoft. (2024). *CREATE TABLE (Transact-SQL)*. [https://learn.microsoft.com/en-us/sql/t-sql/statements/create-table-transact-sql](https://learn.microsoft.com/en-us/sql/t-sql/statements/create-table-transact-sql)

3. Microsoft. (2024). *ALTER TABLE (Transact-SQL)*. [https://learn.microsoft.com/en-us/sql/t-sql/statements/alter-table-transact-sql](https://learn.microsoft.com/en-us/sql/t-sql/statements/alter-table-transact-sql)

4. Microsoft. (2024). *INSERT (Transact-SQL)*. [https://learn.microsoft.com/en-us/sql/t-sql/statements/insert-transact-sql](https://learn.microsoft.com/en-us/sql/t-sql/statements/insert-transact-sql)

5. Microsoft. (2024). *MERGE (Transact-SQL)*. [https://learn.microsoft.com/en-us/sql/t-sql/statements/merge-transact-sql](https://learn.microsoft.com/en-us/sql/t-sql/statements/merge-transact-sql)

6. Microsoft. (2024). *SELECT INTO (Transact-SQL)*. [https://learn.microsoft.com/en-us/sql/t-sql/queries/select-into-clause-transact-sql](https://learn.microsoft.com/en-us/sql/t-sql/queries/select-into-clause-transact-sql)

7. Microsoft. (2024). *CREATE INDEX (Transact-SQL)*. [https://learn.microsoft.com/en-us/sql/t-sql/statements/create-index-transact-sql](https://learn.microsoft.com/en-us/sql/t-sql/statements/create-index-transact-sql)

8. Dolinger, P. (2027). *ETL for Business Intelligence: A practical guide to data provisioning, dimensional modelling, and pipeline design*. NSCC Institute of Technology. CC BY 4.0. — The companion textbook covering SSIS implementation of the concepts introduced in this chapter.

---

*Previous chapter: [Chapter 8 — Advanced Queries: Window Functions, UDFs, and Stored Procedures](../chapter-08-advanced-queries/README.md)*

*Return to: [SQL for Analysts — Table of Contents](../README.md)*

---

> **SQL for Analysts** | © Patrick Dolinger, NSCC Institute of Technology
> [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — Share and adapt freely with attribution
>
> *Thank you for reading. If you use or adapt this material, please attribute as:*
> *Dolinger, P. (2026). SQL for Analysts: A practical introduction to querying, transforming, and understanding relational data. NSCC Institute of Technology. CC BY 4.0.*
