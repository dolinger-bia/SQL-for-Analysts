# Chapter 1: The Relational World — Databases, Tables, and Your First Query

> **SQL for Analysts**
> *A practical introduction to querying, transforming, and understanding relational data*
>
> © Patrick Dolinger, NSCC Institute of Technology
> Licensed under [Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/)
> You are free to share and adapt this material for any purpose, provided appropriate credit is given.

---

## Chapter Overview

This chapter is your entry point into the world of relational databases and SQL. If you have never written a line of SQL before, welcome — this is the right place to start. If you have some prior exposure, the early sections will feel familiar and the later ones will add precision to concepts you may have used intuitively.

By the end of this chapter you will have connected to a live database, navigated its structure, understood what a relational database actually is and why it works the way it does, and written your first SELECT statements. More importantly, you will understand *why* each piece of syntax exists — not just what to type, but what you are asking for when you type it.

By the end of this chapter you will be able to:

- Explain what a relational database is and how it differs from a spreadsheet
- Navigate the CabotTrail Outdoor database environment in SSMS
- Describe the purpose of schemas and explain how they organise tables
- Write a basic SELECT statement that returns specific columns from a table
- Control the number and order of rows returned using TOP and ORDER BY
- Write meaningful comments in SQL code
- Identify the four categories of SQL: DQL, DML, DDL, and DCL

---

## Table of Contents

1. [What Is a Database — and Why Not Just Use a Spreadsheet?](#1-what-is-a-database-and-why-not-just-use-a-spreadsheet)
2. [Relational Databases: The Core Idea](#2-relational-databases-the-core-idea)
3. [SQL: The Language of Data](#3-sql-the-language-of-data)
4. [Connecting to SQL Server with SSMS](#4-connecting-to-sql-server-with-ssms)
5. [The CabotTrail Outdoor Environment](#5-the-cabottrail-outdoor-environment)
6. [Schemas: Organising Tables into Groups](#6-schemas-organising-tables-into-groups)
7. [Your First SELECT Statement](#7-your-first-select-statement)
8. [Controlling Output: TOP and ORDER BY](#8-controlling-output-top-and-order-by)
9. [Writing Comments in SQL](#9-writing-comments-in-sql)
10. [The System Catalog: Asking the Database About Itself](#10-the-system-catalog-asking-the-database-about-itself)
11. [Chapter Summary](#11-chapter-summary)
12. [Review Questions](#12-review-questions)
13. [🔍 Deeper Dive](#-deeper-dive)

---

## 1. What Is a Database — and Why Not Just Use a Spreadsheet?

If you have worked with data before, you have almost certainly used a spreadsheet — Excel, Google Sheets, or something similar. Spreadsheets are excellent tools. They are visual, immediate, and flexible. You can type numbers, write formulas, and see results without learning anything formal.

So why use a database at all?

The answer is scale, structure, and integrity — and understanding each one will shape how you think about data for the rest of your career.

### 1.1 Scale

A spreadsheet comfortably handles thousands of rows. With some patience, it can handle tens of thousands. Beyond that, performance degrades, files become unwieldy, and sharing becomes a problem.

CabotTrail Outdoor's sales data has over 16,000 invoice lines across four years of business. That is a manageable spreadsheet — today. Add another year and it doubles. Add more product lines and customers, and within a few years you have hundreds of thousands of rows. A database handles this without breaking a sweat. SQL queries against tables with millions of rows execute in seconds.

### 1.2 Structure

A spreadsheet is a flat grid. Every piece of information lives in a cell, and there is no formal relationship between cells except what you create with formulas. If a customer's address changes, you find every row where that customer appears and update it manually — or use Find and Replace and hope nothing goes wrong.

A relational database stores *related* information in *related tables*. A customer's address is stored exactly once. Every transaction that references that customer links to the customer record — it does not copy the address. When the address changes, you update it in one place and every transaction instantly reflects the new value.

This is the relational model: organized, precise, and non-redundant.

### 1.3 Integrity

Spreadsheets enforce no rules. You can type text in a column meant for numbers, enter a date that does not exist, or delete a row that other rows depend on — nothing stops you. The result is data corruption that may not be obvious until it matters.

Databases enforce **constraints** — rules that data must satisfy before it can be stored. A date column cannot accept text. A quantity column cannot be negative if the business rule says so. A transaction cannot reference a customer that does not exist. The database is the last line of defence against bad data entering the system.

### 1.4 Concurrency

A spreadsheet is a file. If two people open the same file simultaneously, one will overwrite the other's changes. Coordinating access requires awkward workarounds — emailing files, shared network drives with locking, manual merging.

A database is designed for simultaneous access by many users. Thousands of people can query and update the same database at the same time without overwriting each other's work. The database manages this through a system of locks and transactions that is invisible to most users.

---

## 2. Relational Databases: The Core Idea

A **relational database** organises data into **tables** — structured grids with named columns and typed rows. The "relational" in the name does not mean the tables are friendly with each other — it comes from the mathematical concept of a *relation*, which is essentially what we call a table.

### 2.1 Tables, Rows, and Columns

Every table in a relational database has:

- **Columns** (also called *fields* or *attributes*): the named categories of data in the table. Every column has a data type — text, number, date, etc.
- **Rows** (also called *records* or *tuples*): the individual entries in the table. Every row in the same table has the same columns.
- **A primary key**: one or more columns whose value uniquely identifies each row. No two rows in the same table can have the same primary key value.

This structure is rigid by design. Every row in the `Products` table has exactly the same columns as every other row. You cannot add extra columns for some products and not others. This rigidity is what makes SQL queries fast and predictable.

### 2.2 Relationships Between Tables

The power of the relational model comes from linking tables together. A **foreign key** is a column in one table that refers to the primary key of another table.

Consider CabotTrail's sales data:

- The `Sales.Orders` table has a `CustomerID` column
- `CustomerID` is the primary key of the `Sales.Customers` table
- So `Orders.CustomerID` is a *foreign key* — it links each order to the customer who placed it

This linkage means you do not store the customer's name on every order row. You store the customer's ID. When you want the name, you ask the database to *join* the two tables together.

This is why the relational model is so efficient: information is stored once and referenced everywhere. Changes propagate automatically.

### 2.3 The Difference Between a Database and a Schema and a Table

These three words are often used loosely, but they have precise meanings:

| Term | What it is | CabotTrail example |
|---|---|---|
| **Database** | A named collection of schemas, tables, and other objects, stored in SQL Server | `CabotTrailOutdoorsSales` |
| **Schema** | A namespace that groups related tables within a database | `fact`, `dim` |
| **Table** | A structured grid of columns and rows within a schema | `fact.Sales`, `dim.Customer` |

When you refer to a table fully, you can use three-part notation: `DatabaseName.SchemaName.TableName`. For example: `CabotTrailOutdoorsSales.fact.Sales`.

In practice, once you are working within a specific database (using the `USE` command), you only need `SchemaName.TableName`.

---

## 3. SQL: The Language of Data

**SQL** stands for **Structured Query Language**. It is the standard language for interacting with relational databases. Nearly every relational database system — SQL Server, MySQL, PostgreSQL, Oracle, SQLite, Snowflake, BigQuery — understands SQL. The syntax varies in small ways between systems, but the core concepts are universal.

SQL is not a general-purpose programming language like Python or Java. It does not have loops, functions in the traditional sense, or graphical output. What it does, it does with remarkable precision and efficiency: it asks questions of data.

### 3.1 The Four Categories of SQL

SQL statements are divided into four categories based on what they do:

| Category | Abbreviation | Purpose | Key statements |
|---|---|---|---|
| **Data Query Language** | DQL | Read and retrieve data | `SELECT` |
| **Data Manipulation Language** | DML | Add, change, or remove data | `INSERT`, `UPDATE`, `DELETE` |
| **Data Definition Language** | DDL | Create or modify database structures | `CREATE`, `ALTER`, `DROP` |
| **Data Control Language** | DCL | Manage access and permissions | `GRANT`, `REVOKE` |

**This course focuses primarily on DQL.** The SELECT statement is the tool of the analyst — it lets you ask any question of the data without changing anything. You will touch DML and DDL in later chapters, but the vast majority of your SQL career as an analyst will be writing SELECT statements.

### 3.2 SQL Is Declarative

Most programming languages are *imperative* — you tell the computer *how* to do something, step by step. SQL is *declarative* — you tell the database *what* you want, and the database decides how to retrieve it.

When you write:
```sql
SELECT ProductName, UnitPrice FROM dim.Product WHERE CategoryName = 'Sleeping';
```

You are not telling SQL Server how to find these rows — whether to scan the entire table, use an index, or some other method. You are declaring what you want: the product name and unit price for products in the Sleeping category. The database engine figures out the most efficient way to deliver it.

This is both a strength and, occasionally, a source of confusion. Understanding *why* a query returns what it returns requires understanding the relational model — not just the syntax.

### 3.3 SQL Is Not Case-Sensitive (Mostly)

SQL keywords (`SELECT`, `FROM`, `WHERE`) are not case-sensitive in SQL Server. `select`, `SELECT`, and `Select` all work. Convention is to write keywords in uppercase to distinguish them from table and column names — but this is style, not requirement.

String comparisons *are* case-sensitive in some database configurations. SQL Server's default collation is case-insensitive (`Latin1_General_CI_AS` — CI = Case Insensitive), which means `WHERE CategoryName = 'sleeping'` and `WHERE CategoryName = 'Sleeping'` return the same results. Other databases may behave differently. When in doubt, match the case of the data.

---

## 4. Connecting to SQL Server with SSMS

**SQL Server Management Studio (SSMS)** is the graphical tool for working with SQL Server. It provides a visual interface for connecting to databases, writing and running SQL queries, and exploring database objects.

### 4.1 Opening a Connection

When SSMS opens, a **Connect to Server** dialog appears. Fill in:

- **Server type:** Database Engine
- **Server name:** *(provided by your instructor — typically a server name or `localhost`)*
- **Authentication:** Windows Authentication *(uses your Windows login; no separate password required)*

Click **Connect**. If successful, the **Object Explorer** panel on the left populates with a tree showing all available databases.

> **Troubleshooting:** If the connection fails, check that the server name is correct and that SQL Server is running. Your instructor can verify this. A common mistake is leaving the server name blank or typing `localhost` when the server is on a different machine.

### 4.2 Navigating Object Explorer

The Object Explorer shows the database hierarchy. Expand nodes by clicking the triangle (▶) beside them:

```
[Server name]
  └── Databases
        └── CabotTrailOutdoorsSales
              └── Tables
                    ├── dim.Calendar
                    ├── dim.Customer
                    ├── dim.DeliveryMethod
                    ├── dim.Employee
                    ├── dim.Product
                    └── fact.Sales
```

**Right-clicking** on any table gives you useful options:
- **Select Top 1000 Rows** — opens a query window pre-populated with a SELECT query
- **Design** — shows the table's column structure, data types, and constraints (read-only in most environments)
- **Properties** — metadata about the table (row count, creation date, etc.)

### 4.3 The Query Window

The **Query Window** is where you write and run SQL. Open a new one with **File → New → Database Engine Query** or by pressing `Ctrl+N`.

Key controls:
- **Execute** button (▶) or `F5` — runs the query
- **Parse** button or `Ctrl+F5` — checks syntax without running
- **Results pane** — appears below the query window after execution
- **Messages pane** — shows row counts, warnings, and errors

**Important:** Always set the active database at the top of the query window using `USE`:

```sql
USE CabotTrailOutdoorsSales;
```

This tells SQL Server which database you are working in. If you forget, queries may fail with "Invalid object name" errors because SQL Server is looking in the wrong database.

---

## 5. The CabotTrail Outdoor Environment

CabotTrail Outdoor is a fictional outdoor gear retailer based in Nova Scotia, Canada. It sells products across thirteen categories — Shelter, Sleeping, Footwear, Accessories, Navigation, and more — to wholesale and retail customers across Canada.

You will use CabotTrail's data throughout this course and the two courses that follow. By the time you finish DBAS 2103, you will know this business's data better than most of its fictional employees.

### 5.1 The Learning Environment

The databases are designed to support learning:

- **Varied data:** Revenue varies meaningfully across products, categories, and territories. Margin rates differ by category. Returns have realistic reasons and refund percentages. This means analytical queries produce interesting, non-uniform results.
- **Realistic size:** 16,359 sales invoice lines across 4+ years. Large enough to see trends; small enough to understand every query result.
- **Progressive complexity:** The Sales datamart has six tables. The full data warehouse has dozens. You will start simple and build up.

### 5.2 The Sales Data Mart: Your Starting Database

For the first half of this course, you will work in `CabotTrailOutdoorsSales` — the Sales data mart. It has exactly six tables:

| Table | What it contains | Rows |
|---|---|---|
| `fact.Sales` | One row per product on each customer invoice | 16,359 |
| `dim.Calendar` | One row per calendar day | 4,017 |
| `dim.Customer` | One row per customer | 100 |
| `dim.Product` | One row per product | 142 |
| `dim.Employee` | One row per employee | 50 |
| `dim.DeliveryMethod` | One row per delivery method | 12 |

This is a **star schema** — a dimensional data model where a central fact table (`fact.Sales`) is surrounded by dimension tables that provide context. The diagram looks like a star:

```
                    dim.Calendar
                         │
dim.Customer ────── fact.Sales ────── dim.Product
                         │
              dim.Employee ── dim.DeliveryMethod
```

You will learn why this structure exists, and how to query it effectively, throughout this course. For now, the key insight is: `fact.Sales` contains the measurements (quantities, prices, totals), and the dimension tables contain the descriptions (who, what, when, how).

### 5.3 Exploring the Tables in SSMS

Before writing any queries, spend a few minutes exploring. Right-click each table and choose **Select Top 1000 Rows**. Observe:

- `dim.Product` is human-readable on its own — you can see product names, categories, and prices immediately
- `dim.Customer` shows customer names, territories, and credit information
- `fact.Sales` is full of numbers and IDs — it is not readable without the dimension tables

This observation is intentional. The fact table stores raw measurements. The dimension tables give those measurements meaning. Together, they answer business questions.

---

## 6. Schemas: Organising Tables into Groups

You have noticed that every table name in `CabotTrailOutdoorsSales` has a prefix: `fact.` or `dim.`. This prefix is the **schema** — a namespace that groups related tables.

### 6.1 What a Schema Does

A schema serves two purposes:

**Organisation:** Tables that serve similar purposes are grouped together. In `CabotTrailOutdoorsSales`, all dimension tables are in the `dim` schema and the fact table is in the `fact` schema. In the full CabotTrail OLTP database (which you will meet in Chapter 9), tables are grouped by business function: `Sales`, `Purchasing`, `Inventory`, `Application`.

**Security:** Permissions can be granted at the schema level. An analyst might be given read access to the `fact` and `dim` schemas but not to the `ETL` schema (which contains internal pipeline tables).

### 6.2 Referencing Tables with Schema Names

Always include the schema name when referencing a table:

```sql
-- Correct: schema-qualified name
SELECT ProductName FROM dim.Product;

-- Risky: unqualified name (SQL Server will look in the default schema, usually dbo)
SELECT ProductName FROM Product;  -- May fail or return the wrong table
```

The habit of schema-qualifying table names prevents subtle bugs, especially in databases with many schemas where the same table name might exist in more than one.

---

## 7. Your First SELECT Statement

The `SELECT` statement is the foundation of all SQL querying. Everything you do as a data analyst starts with `SELECT`.

### 7.1 The Basic Structure

```sql
SELECT  column1, column2, column3
FROM    schema.TableName;
```

Two clauses are required:
- `SELECT` — specifies which columns to return
- `FROM` — specifies which table to read from

The semicolon (`;`) marks the end of a statement. In SQL Server it is technically optional for single statements, but it is good practice to include it — especially when writing scripts with multiple statements.

### 7.2 Your First Query

```sql
-- Connect to the Sales data mart
USE CabotTrailOutdoorsSales;

-- Return three columns from the Product dimension
SELECT  ProductID,
        ProductName,
        CategoryName
FROM    dim.Product;
```

Run this query by pressing F5. You should see 142 rows in the Results pane — one for each CabotTrail product.

**Reading the result:** Each row is one product. The three columns show the product's ID, name, and category. SQL Server returns the rows in no guaranteed order unless you specify one (covered in section 8.2).

### 7.3 Selecting All Columns with *

The asterisk (`*`) is shorthand for "all columns":

```sql
SELECT  *
FROM    dim.Product;
```

This returns every column in the table. Try it — you will see many more columns than the three in the previous query.

**When to use `*`:** For exploration — when you want to see what columns a table has. Not for production queries or submitted work. Reasons:

1. You often do not need all columns. Returning columns you do not use wastes memory and makes results harder to read.
2. If someone adds a column to the table later, your `SELECT *` query silently starts returning it. This can break downstream code that expected a specific column count.
3. It communicates nothing about what you wanted. `SELECT CustomerName, SalesTerritory FROM dim.Customer` is clear; `SELECT * FROM dim.Customer` is vague.

**Good practice:** Name your columns explicitly. Use `SELECT *` only when exploring a new table for the first time.

### 7.4 Column Formatting and Readability

SQL Server ignores extra whitespace. These two queries are identical:

```sql
SELECT ProductID, ProductName, CategoryName FROM dim.Product;
```

```sql
SELECT  ProductID,
        ProductName,
        CategoryName
FROM    dim.Product;
```

The second is easier to read. Professional SQL aligns keywords and column names vertically, making queries scannable at a glance. When queries grow to 20 or 30 lines, this consistency matters enormously. Adopt the habit now.

### 7.5 Changing the Active Database with USE

The `USE` statement sets the active database for the current session. Everything after it applies to that database until you `USE` a different one:

```sql
USE CabotTrailOutdoorsSales;

-- This query runs against CabotTrailOutdoorsSales
SELECT CustomerName FROM dim.Customer;

USE CabotTrailOutdoorDW;

-- This query now runs against CabotTrailOutdoorDW
SELECT CustomerName FROM Dimension.DimCustomer;
```

Alternatively, you can use **three-part naming** to reference a table in any database without switching:

```sql
-- Three-part name: DatabaseName.SchemaName.TableName
SELECT CustomerName
FROM   CabotTrailOutdoorsSales.dim.Customer;
```

Three-part naming is useful when a query needs data from multiple databases simultaneously. You will use it extensively in later chapters.

---

## 8. Controlling Output: TOP and ORDER BY

A raw `SELECT` returns all rows in no guaranteed order. Two clauses let you control this: `TOP` limits how many rows are returned, and `ORDER BY` determines their sequence.

### 8.1 Limiting Rows with TOP

`TOP n` returns only the first *n* rows from the result set. It always works together with `ORDER BY` — otherwise, which rows are "first" is undefined:

```sql
-- The 5 most expensive products
SELECT  TOP 5
        ProductName,
        CategoryName,
        UnitPrice
FROM    dim.Product
ORDER BY UnitPrice DESC;
```

`TOP` is useful for:
- **Exploratory queries:** See a sample of the data without loading all rows
- **"Best of" questions:** Top 10 customers by revenue, top 5 products by return rate
- **Debugging:** Check whether a query returns the right structure before running it against millions of rows

### 8.2 Sorting Results with ORDER BY

`ORDER BY` sorts the result set by one or more columns. It always appears last in the query:

```sql
-- Products sorted alphabetically by name
SELECT  ProductName,
        CategoryName,
        UnitPrice
FROM    dim.Product
ORDER BY ProductName ASC;    -- ASC = ascending (A to Z, 0 to 9). This is the default.
```

```sql
-- Products sorted by price from most to least expensive
SELECT  ProductName,
        CategoryName,
        UnitPrice
FROM    dim.Product
ORDER BY UnitPrice DESC;     -- DESC = descending (Z to A, highest to lowest)
```

**Sorting by multiple columns:** List them in priority order, separated by commas:

```sql
-- Products sorted by category (A to Z), then by price within each category (high to low)
SELECT  ProductName,
        CategoryName,
        UnitPrice
FROM    dim.Product
ORDER BY CategoryName ASC,
         UnitPrice    DESC;
```

This sorts the entire result set by `CategoryName` first. Where two products share the same category, they are further sorted by `UnitPrice` descending.

**Sorting by column position:** You can use a column's position number instead of its name:

```sql
ORDER BY 2 ASC, 3 DESC;  -- Sort by the 2nd column ascending, then 3rd descending
```

This works but is fragile — if you reorder the columns in the SELECT list, the sort changes unexpectedly. Prefer using column names.

### 8.3 ORDER BY Without TOP

`ORDER BY` without `TOP` sorts all rows in the result:

```sql
-- All 50 employees sorted alphabetically by full name
SELECT  FullName,
        JobTitle,
        Department
FROM    dim.Employee
ORDER BY FullName ASC;
```

**Note on performance:** Sorting is an expensive operation on large tables. SQL Server must read all qualifying rows, then sort them, before returning results. For exploratory queries against small tables (like CabotTrail's dimension tables), this is instant. Against large fact tables, an `ORDER BY` without appropriate indexes can be slow. You will return to this topic in Chapter 5.

### 8.4 TOP With Ties

A subtle issue: what if the 5th and 6th products have exactly the same `UnitPrice`? `TOP 5` returns 5 rows — but which one it excludes between two tied values is arbitrary. `TOP 5 WITH TIES` returns all rows that tie for the last position:

```sql
-- Top 5 most expensive products — include all products that tie for 5th place
SELECT  TOP 5 WITH TIES
        ProductName,
        UnitPrice
FROM    dim.Product
ORDER BY UnitPrice DESC;
-- May return more than 5 rows if there are ties at the 5th price level
```

---

## 9. Writing Comments in SQL

**Comments** are text in your SQL code that the database ignores. They exist for the human reading the code — explaining what a query does, why a decision was made, or what a particular expression means.

### 9.1 Single-Line Comments

A single-line comment starts with two dashes (`--`). Everything from the `--` to the end of the line is ignored:

```sql
-- Return all products in the Sleeping category
SELECT  ProductName,
        UnitPrice           -- Price before tax
FROM    dim.Product
WHERE   CategoryName = 'Sleeping';  -- Filter to one category
```

### 9.2 Multi-Line Comments

A multi-line comment is wrapped in `/* */`. Useful for longer explanations or for temporarily disabling blocks of SQL:

```sql
/*
   This query answers the business question:
   "What are our most expensive products in each category?"
   
   Written by: Patrick Dolinger
   Date: 2026-09-10
   Updated: Added UnitPrice column
*/
SELECT  TOP 10
        ProductName,
        CategoryName,
        UnitPrice
FROM    dim.Product
ORDER BY UnitPrice DESC;
```

### 9.3 When to Comment

Comments are not optional decoration — they are professional practice. Every submitted lab and production query should have:

1. A brief description of what business question the query answers
2. Comments on any non-obvious expressions or decisions
3. Your name and date for long scripts

A query without a comment says "I wrote this and understood it at the time." A query with good comments says "I wrote this, understood it, and want the next person to understand it too."

---

## 10. The System Catalog: Asking the Database About Itself

SQL Server stores information about its own structure — table names, column names, data types, constraints — in a set of system views. You can query these views just like regular tables. This is called the **system catalog** or **information schema**.

### 10.1 Listing Tables in a Database

```sql
-- What tables exist in the current database?
USE CabotTrailOutdoorsSales;

SELECT  TABLE_SCHEMA,
        TABLE_NAME,
        TABLE_TYPE
FROM    INFORMATION_SCHEMA.TABLES
ORDER BY TABLE_SCHEMA, TABLE_NAME;
```

`INFORMATION_SCHEMA.TABLES` is a system view that returns one row for each table and view in the database. `TABLE_SCHEMA` is the schema name; `TABLE_NAME` is the table name; `TABLE_TYPE` is either 'BASE TABLE' (a regular table) or 'VIEW'.

### 10.2 Listing Columns in a Table

```sql
-- What columns does fact.Sales have?
SELECT  COLUMN_NAME,
        DATA_TYPE,
        CHARACTER_MAXIMUM_LENGTH,
        NUMERIC_PRECISION,
        NUMERIC_SCALE,
        IS_NULLABLE
FROM    INFORMATION_SCHEMA.COLUMNS
WHERE   TABLE_SCHEMA = 'fact'
AND     TABLE_NAME   = 'Sales'
ORDER BY ORDINAL_POSITION;
```

This query is enormously useful when you encounter a new table and want to understand its structure without right-clicking through SSMS menus. You will use variations of it throughout this course.

### 10.3 Why the System Catalog Matters

As an analyst, you will regularly work with databases you did not design and have not seen before. The system catalog lets you explore unfamiliar databases systematically:

- Which tables exist and which schemas they belong to?
- What columns does this table have and what are their data types?
- Which columns allow NULL values (meaning the column might be empty)?

These are the first questions you ask about any new table. The system catalog answers them.

---

## 11. Chapter Summary

- A **relational database** stores data in structured tables with named columns and typed rows. Related tables are linked through primary and foreign keys — data is stored once and referenced many times.

- Databases offer **scale** (millions of rows), **structure** (rigid schemas), **integrity** (enforced constraints), and **concurrency** (simultaneous access) that spreadsheets cannot match.

- **SQL** (Structured Query Language) is the standard language for interacting with relational databases. It is declarative — you specify what you want, not how to get it. This course focuses on **DQL** (Data Query Language), centred on the `SELECT` statement.

- **SSMS** (SQL Server Management Studio) is the tool for connecting to and querying SQL Server. Use `USE DatabaseName` to set the active database; reference tables as `schema.TableName`.

- The **CabotTrail Outdoor** Sales data mart (`CabotTrailOutdoorsSales`) has six tables. The central `fact.Sales` table holds measurements; the five dimension tables (`dim.Calendar`, `dim.Customer`, `dim.Product`, `dim.Employee`, `dim.DeliveryMethod`) provide context.

- A **SELECT statement** requires at minimum `SELECT` (what columns) and `FROM` (what table). Add `TOP n` to limit rows and `ORDER BY` to sort. Use `ORDER BY` with `ASC` (default) or `DESC`.

- **Comments** (`--` for single-line, `/* */` for multi-line) explain intent. They are professional practice, not optional decoration.

- The **system catalog** (`INFORMATION_SCHEMA.TABLES`, `INFORMATION_SCHEMA.COLUMNS`) lets you query the database about its own structure — essential for exploring unfamiliar databases.

---

## 12. Review Questions

1. A colleague suggests storing the company's sales data in Excel because "it's simpler." Describe three specific limitations of Excel that would become problems as the sales data grows, and explain how a relational database addresses each.

2. In `CabotTrailOutdoorsSales`, `fact.Sales` contains a `CustomerKey` column rather than `CustomerName`. Explain why the designer made this choice. What advantage does storing a key rather than a name provide?

3. Write a query that returns the `FullName`, `JobTitle`, and `Department` of all employees in `dim.Employee`, sorted alphabetically by `FullName`.

4. Write a query that returns the top 3 most expensive products from `dim.Product`. Include the `ProductName`, `CategoryName`, and `UnitPrice`. What does `TOP 3 WITH TIES` do differently, and when might you prefer it?

5. A developer writes `SELECT * FROM Sales` and gets an error: "Invalid object name 'Sales'." Explain two separate reasons this error might occur and how to fix each.

6. The `INFORMATION_SCHEMA.COLUMNS` query for `fact.Sales` returns a column called `IS_NULLABLE`. Some columns show 'YES' and others show 'NO'. What does this mean, and why might an analyst care whether a column allows NULL values?

7. Write a query that returns all delivery methods from `dim.DeliveryMethod`, sorted by `DeliveryMethodName` alphabetically. Add a comment above the query explaining what business question it answers.

8. Without running it, predict what this query returns and explain your reasoning:
   ```sql
   SELECT TOP 1 ProductName, UnitPrice
   FROM   dim.Product
   ORDER BY UnitPrice ASC;
   ```

---

## 🔍 Deeper Dive

### Going Further with the Relational Model

#### The History of the Relational Model

The relational model was proposed by **E.F. Codd**, a computer scientist at IBM, in a landmark 1970 paper: *"A Relational Model of Data for Large Shared Data Banks."* Published in the *Communications of the ACM*, it described a mathematical framework — based on set theory and predicate logic — for organising data that would become the foundation of every relational database system built since.

Before Codd's paper, databases were *hierarchical* or *network* models — data was navigated through physical pointers, and queries required knowledge of the database's physical layout. Codd's insight was that data should be described by its logical relationships, not its physical storage, and that queries should be declarative (what you want) rather than navigational (how to find it).

SQL, as the language implementation of the relational model, was standardised by ANSI (American National Standards Institute) in 1986 and ISO in 1987. The current standard, **ISO/IEC 9075**, is updated approximately every four years. SQL Server implements a substantial portion of the standard with its own extensions (T-SQL — Transact-SQL).

For the curious: Codd's original 1970 paper is freely available and surprisingly readable for a foundational computer science paper. It is worth reading to understand why the relational model was such a conceptual breakthrough.

Codd, E. F. (1970). A relational model of data for large shared data banks. *Communications of the ACM*, 13(6), 377–387.

#### SSMS Keyboard Shortcuts Worth Learning

SSMS is a full-featured IDE with many keyboard shortcuts. These are the ones analysts use most often:

| Shortcut | Action |
|---|---|
| `F5` | Execute the query (or selected text) |
| `Ctrl+F5` | Parse the query (check syntax without running) |
| `Ctrl+N` | New query window |
| `Ctrl+K, Ctrl+C` | Comment out selected lines |
| `Ctrl+K, Ctrl+U` | Uncomment selected lines |
| `Ctrl+L` | Display estimated execution plan (not running the query) |
| `Ctrl+M` | Toggle actual execution plan display |
| `Ctrl+R` | Show/hide results pane |
| `Ctrl+Space` | IntelliSense completion (suggest column/table names) |
| `Alt+F1` | Show object properties for selected object name |

Investing 20 minutes learning these shortcuts saves hours over a course. Start with F5, Ctrl+K/C, and Ctrl+K/U — the most used by far.

#### T-SQL vs Standard SQL

This book uses **T-SQL** (Transact-SQL) — Microsoft's implementation of SQL with extensions specific to SQL Server. Most of what you learn here is standard SQL and will transfer directly to other database systems. A few T-SQL-specific features to be aware of:

| Feature | T-SQL syntax | Standard SQL | Notes |
|---|---|---|---|
| `TOP n` | `SELECT TOP 10 ...` | `FETCH FIRST 10 ROWS ONLY` | PostgreSQL/Oracle/MySQL use LIMIT |
| Date functions | `GETDATE()` | `CURRENT_TIMESTAMP` | Most platforms support CURRENT_TIMESTAMP |
| String functions | `LEN()` | `LENGTH()` | PostgreSQL, MySQL use LENGTH |
| Identity columns | `INT IDENTITY(1,1)` | `INT GENERATED ALWAYS AS IDENTITY` | Standard SQL:2003 syntax |

When you encounter a different database platform — Snowflake, PostgreSQL, BigQuery — your SQL knowledge transfers directly. You will need to look up syntax differences for a handful of functions, but the SELECT, WHERE, JOIN, and GROUP BY foundations are identical everywhere.

Microsoft's T-SQL documentation is comprehensive and freely available:
[Transact-SQL Reference — Microsoft Learn](https://learn.microsoft.com/en-us/sql/t-sql/language-reference)

#### What Is a Star Schema?

The CabotTrail Sales data mart uses a **star schema** — a specific design pattern for analytical databases developed by Ralph Kimball in the 1990s. It is called a star schema because of its visual appearance: a central fact table surrounded by dimension tables, like the spokes of a star.

Star schemas are designed for *reading*, not writing. The same data exists in multiple places (a customer's name appears on every order, conceptually) to make queries simple and fast. This is the opposite design philosophy from a normalized OLTP database, which stores each piece of information exactly once.

You will encounter star schemas constantly as a data analyst — they power most BI tools, most dashboards, and most analytical databases. Understanding them deeply is one of the most valuable things you can do in this course.

The full explanation of dimensional modelling — why star schemas exist and how to design them — is the subject of the companion textbook *ETL for Business Intelligence*, Chapter 2.

---

### Industry Perspectives

#### SQL in the Analyst's Toolkit

A 2023 survey of data professionals by Stack Overflow found SQL to be the most widely used language in data science and analytics work — more common than Python, R, or any other tool. This is not because SQL is the most powerful language (Python is far more flexible) but because SQL is where the data *is*.

Data lives in relational databases. Analytical results need to be communicated in terms of that data. SQL is the lingua franca that connects analysts to data, regardless of what BI tool, programming language, or platform sits on top.

Learning SQL well is arguably the highest-return investment a data analyst can make.

#### SQL Server and the Microsoft Data Platform

SQL Server is one of three dominant relational database platforms alongside Oracle Database and PostgreSQL. In business intelligence environments — particularly in organizations already invested in Microsoft products — SQL Server is the most common choice. It integrates natively with Excel (via Power Query and Power Pivot), Power BI, and Azure.

Microsoft's cloud-native analytical platform, **Azure Synapse Analytics**, extends SQL Server concepts to cloud-scale data warehousing — the same T-SQL syntax applies to tables with billions of rows. The skills you develop in this course transfer directly to the cloud platform.

[SQL Server Technical Documentation](https://learn.microsoft.com/en-us/sql/sql-server/)

---

### References and Further Reading

1. Codd, E. F. (1970). A relational model of data for large shared data banks. *Communications of the ACM*, 13(6), 377–387. — The foundational paper. Remarkably accessible.

2. Date, C. J. (2019). *Database Design and Relational Theory: Normal Forms and All That Jazz* (2nd ed.). Apress. — A rigorous treatment of the relational model for those who want the mathematical foundations.

3. Vieira, R. (2021). *Beginning SQL Server: A Beginner's Guide* (5th ed.). Apress. — Comprehensive beginner-to-intermediate coverage of SQL Server.

4. Microsoft. (2024). *Transact-SQL Reference*. [https://learn.microsoft.com/en-us/sql/t-sql/language-reference](https://learn.microsoft.com/en-us/sql/t-sql/language-reference)

5. Microsoft. (2024). *SQL Server Management Studio documentation*. [https://learn.microsoft.com/en-us/sql/ssms/sql-server-management-studio-ssms](https://learn.microsoft.com/en-us/sql/ssms/sql-server-management-studio-ssms)

6. Microsoft. (2024). *INFORMATION_SCHEMA Views*. [https://learn.microsoft.com/en-us/sql/relational-databases/system-information-schema-views/system-information-schema-views-transact-sql](https://learn.microsoft.com/en-us/sql/relational-databases/system-information-schema-views/system-information-schema-views-transact-sql)

7. Stack Overflow. (2023). *Developer Survey 2023 — Most popular technologies*. [https://survey.stackoverflow.co/2023/](https://survey.stackoverflow.co/2023/)

---

*Next chapter: [Chapter 2 — Filtering and Sorting: Asking Precise Questions](../chapter-02-filtering-and-sorting/README.md)*

---

> **SQL for Analysts** | © Patrick Dolinger, NSCC Institute of Technology
> [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — Share and adapt freely with attribution
