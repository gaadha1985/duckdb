# DuckDB Codebase Explained — For Someone Who Has Never Coded

---

## What Is DuckDB, and Why Does It Exist?

Imagine you have a massive spreadsheet — millions of rows of sales data, medical records, or financial transactions — and you need to answer questions like:

- "What were the total sales in each region last year?"
- "Which customers bought more than three products?"
- "What is the average age of patients who responded to treatment?"

A spreadsheet program like Excel struggles with millions of rows. A traditional database designed for websites (like MySQL or PostgreSQL) is optimized for looking up *individual records fast*, not for crunching large volumes of numbers. That is where DuckDB fits in.

**DuckDB is a database engine built specifically for analytical questions** — questions that scan, calculate, and summarize large amounts of data. It is designed to run *inside* your program (in-process), meaning there is no separate server to install or maintain. You just include DuckDB in your application and it works immediately.

---

## A Helpful Analogy: DuckDB as a Library

Think of DuckDB like a very sophisticated library for data:

- You (the user) walk in and ask a question in **SQL** (a standard language for asking databases questions).
- A **librarian** (the parser) reads your question and figures out what you are really asking.
- A **research assistant** (the planner) decides the most efficient way to find the answer.
- A **team of workers** (the executor) fan out and actually retrieve and calculate the results.
- The **filing system** (storage) is how all the data books are organized on the shelves.

Every piece of the DuckDB codebase corresponds to one of these roles.

---

## The Big Picture: What Happens When You Ask a Question

When you type a SQL question like:

```sql
SELECT region, SUM(sales) FROM orders GROUP BY region
```

DuckDB processes it through a series of stages, like an assembly line:

```
Your SQL text
     ↓
1. PARSER         — reads the text and understands its structure
     ↓
2. PLANNER        — creates a plan for answering the question
     ↓
3. OPTIMIZER      — finds a smarter, faster way to execute the plan
     ↓
4. EXECUTOR       — actually runs the plan and crunches the numbers
     ↓
5. STORAGE        — reads data from disk when needed
     ↓
Your answer
```

Each of these stages lives in its own folder inside the codebase.

---

## The Folder Structure — A Tour of the Building

```
/duckdb
├── src/           ← The brain and muscle of DuckDB
├── extension/     ← Optional add-on features
├── test/          ← Thousands of checks to make sure nothing is broken
├── benchmark/     ← Speed tests
├── tools/         ← Helpers for Python, R, Java, and other languages
├── third_party/   ← External libraries DuckDB borrows from
├── scripts/       ← Automation tools for developers
└── data/          ← Sample data files
```

---

## Inside `src/` — The Heart of DuckDB

This is where the real code lives. Each subdirectory is responsible for one part of the assembly line.

### `src/main/` — The Front Door

These files are the entry point. When another program (like a Python script) wants to use DuckDB, it starts here.

- **`database.cpp`** — Creates and manages a DuckDB database instance. Think of this as unlocking and opening the library.
- **`client_context.cpp`** — Represents one user's session (connection). Each person using the database gets their own context, tracking what they are doing, their settings, and their progress.

### `src/parser/` — The Librarian Who Reads Your Question

SQL is text. Computers need structure, not text. The parser's job is to convert a SQL string like `SELECT name FROM people WHERE age > 30` into a structured object (called an **Abstract Syntax Tree**, or AST) that later stages can work with.

Think of it like diagramming a sentence in English class — breaking "The quick brown fox jumps" into subject, verb, and object. The parser does the same for SQL.

- It uses a technique called **PEG (Parsing Expression Grammar)** — a formal set of rules describing what valid SQL looks like.
- If your SQL has a typo or invalid syntax, the parser catches it here and reports an error.

### `src/planner/` — The Research Assistant Who Makes a Plan

Once the parser has understood your question, the planner figures out *how* to answer it. This involves two things:

1. **Binding** — Matching the names in your query (like `orders` or `region`) to actual tables and columns in the database. If you ask for a table that does not exist, the error comes from here.
2. **Logical Planning** — Creating a step-by-step logical plan. For example: "First filter rows where date > 2023, then group by region, then sum the sales column."

The result is a **Logical Plan** — a tree of operations describing what needs to happen, without yet worrying about exactly how to make it fast.

### `src/optimizer/` — The Efficiency Expert

The optimizer takes the logical plan and looks for ways to make it faster without changing the answer.

Examples of optimizations:
- **Predicate pushdown**: If you want rows where `age > 30`, filter them out early rather than carrying all rows through every step and filtering at the end.
- **Join reordering**: If you are combining multiple tables, the order in which you combine them can dramatically affect speed. The optimizer finds the best order.
- **Expression simplification**: `1 + 1` can be replaced with `2` at planning time.

The optimizer is like a chess player — it thinks several moves ahead to find the most efficient path.

### `src/execution/` — The Workers Who Do the Actual Work

This is where the plan becomes reality. The execution engine runs the optimized plan and produces actual results.

DuckDB uses **vectorized execution** — instead of processing one row at a time (like reading a book word by word), it processes thousands of rows at once in batches (like reading a whole page at once). This is far faster on modern computer hardware because CPUs are very good at doing the same operation on many values simultaneously.

- Each operation (filter, join, aggregate, sort) is implemented as a **physical operator**.
- Operators are connected into **pipelines** that pass data from one to the next.
- Multiple pipelines can run **in parallel** on different CPU cores.

### `src/storage/` — The Filing System

Storage manages how data is actually kept on disk and loaded into memory.

- Data is stored in **blocks** (fixed-size chunks), similar to pages in a book.
- DuckDB uses **columnar storage** — all values from a single column are stored together, rather than storing entire rows together. This is much faster for analytical queries that only need a few columns from millions of rows.
- A **buffer manager** decides which blocks to keep in memory and which to swap out to disk, like a librarian deciding which books to keep on the desk vs. back on the shelf.
- The **WAL (Write-Ahead Log)** ensures that if your computer crashes mid-write, data is not corrupted. Every change is written to a log first.

### `src/catalog/` — The Index and Card Catalog

The catalog is the database's memory of what exists: which tables, columns, indexes, functions, and data types are defined. When the planner asks "does table `orders` exist, and what columns does it have?", the catalog answers.

### `src/transaction/` — The Traffic Controller

Transactions ensure that multiple people using the database at once do not interfere with each other. If two people are updating data simultaneously, the transaction manager ensures neither sees partial results from the other.

DuckDB uses **MVCC (Multi-Version Concurrency Control)** — when someone writes data, old versions are kept temporarily so readers can still see a consistent snapshot. This avoids locking readers out.

### `src/function/` — The Toolbox

This folder contains all the built-in functions available in SQL — things like:
- **Aggregate functions** (`SUM`, `COUNT`, `AVG`, `MIN`, `MAX`) — summarize multiple rows into one value
- **Scalar functions** (`UPPER`, `LENGTH`, `ROUND`, `SUBSTRING`) — transform a single value
- **Window functions** (`RANK`, `LAG`, `LEAD`, `ROW_NUMBER`) — compute values relative to surrounding rows
- **Table functions** — functions that return entire tables (like `read_csv('file.csv')`)
- **Cast functions** — convert between data types (e.g., text to number)

### `src/common/` — Shared Utilities

This folder contains building blocks used everywhere else — things like:
- Data types (`INTEGER`, `VARCHAR`, `DATE`, `TIMESTAMP`, etc.)
- Mathematical utilities
- String manipulation tools
- File system abstractions
- Memory management helpers

### `src/parallel/` — The Parallelism Manager

Modern computers have many CPU cores. This folder manages splitting work across all available cores so queries run faster. It handles **pipelines**, **task scheduling**, and **thread synchronization**.

---

## `extension/` — Optional Add-Ons

DuckDB is designed to be lean by default but extendable. Extensions are optional modules you can load when needed.

| Extension | What It Adds |
|-----------|--------------|
| `parquet` | Read/write Apache Parquet files (a popular data science format) |
| `json` | Work with JSON data in SQL queries |
| `icu` | International text sorting and comparison |
| `tpch` / `tpcds` | Generate standard industry benchmark datasets |
| `delta` | Support for Delta Lake (a data warehouse format) |
| `autocomplete` | Smart query completion in the CLI |
| `jemalloc` | A faster memory allocator for performance |

Extensions can be loaded at runtime without recompiling DuckDB — like installing a browser extension without reinstalling the browser.

---

## `test/` — The Safety Net

DuckDB has thousands of tests organized by feature. Every time a developer changes something, all tests are re-run automatically to make sure nothing is accidentally broken.

Tests are written in a readable format called **sqllogictest**:

```sql
# Test that basic addition works
query I
SELECT 1 + 1
----
2
```

This says: "Run `SELECT 1 + 1`, and the expected answer is `2`." If DuckDB ever returns something else, the test fails and developers are alerted.

Test folders cover every feature:
- `test/sql/select/` — basic SELECT queries
- `test/sql/join/` — combining tables
- `test/sql/aggregate/` — SUM, COUNT, etc.
- `test/sql/window/` — window functions
- `test/sql/transactions/` — concurrent access
- `test/sql/storage/` — data persistence
- ...and dozens more

---

## `tools/` — DuckDB in Other Languages

DuckDB is written in C++ but can be used from many languages. The `tools/` directory contains the bridges:

- **`pythonpkg/`** — the Python library (`import duckdb`)
- **`shell/`** — the interactive command-line tool
- **`juliapkg/`** — Julia language bindings
- **`rpkg/`** — R language bindings
- **`nodejs/`** — JavaScript/Node.js bindings

These bridges translate between the conventions of each language and DuckDB's internal C++ API.

---

## `third_party/` — Borrowed Libraries

No software is built entirely from scratch. DuckDB uses well-tested open-source libraries for things like:

- **Compression algorithms** (making stored data smaller)
- **Fast number formatting**
- **Unicode handling**
- **Cryptography**
- **Unit testing framework**

Using established libraries means DuckDB's developers can focus on what makes DuckDB unique, rather than reinventing the wheel.

---

## How Data Is Stored — A Deeper Look

Understanding DuckDB's storage model helps explain why it is fast for analytics.

### Row Storage vs. Column Storage

Most traditional databases store data **row by row**:
```
Row 1: [Alice, 30, New York, 50000]
Row 2: [Bob,   25, Chicago,  45000]
Row 3: [Carol, 35, New York, 60000]
```

DuckDB stores data **column by column**:
```
Names:   [Alice, Bob, Carol]
Ages:    [30, 25, 35]
Cities:  [New York, Chicago, New York]
Salaries:[50000, 45000, 60000]
```

If you ask "What is the average salary?", row storage must read every field of every row just to get to the salary. Column storage reads only the salary column — ignoring everything else. For tables with hundreds of columns and millions of rows, this is a massive speedup.

### Compression

Since all values in a column are of the same type (all numbers, or all dates, or all city names), DuckDB can compress them very efficiently. For example, "New York" repeated thousands of times can be stored as a code like `1`, saving space and making reads faster.

---

## How Queries Run in Parallel

Modern computers have 4, 8, 16, or more CPU cores. DuckDB tries to use all of them.

Imagine a query that needs to scan 100 million rows and compute a sum. DuckDB splits the work:

- Core 1 processes rows 1–25 million
- Core 2 processes rows 25–50 million
- Core 3 processes rows 50–75 million
- Core 4 processes rows 75–100 million
- Finally, the four partial sums are combined

This is called **parallel execution**, and it is why DuckDB can often answer complex questions on large datasets in seconds rather than minutes.

---

## The Developer Workflow

When a developer wants to add a new feature or fix a bug, they follow this flow:

1. **Understand** — Read the existing code and tests to understand how things work.
2. **Write code** — Make changes in the appropriate `src/` subfolder.
3. **Write tests** — Add a `.test` file in `test/sql/` to verify the new behavior.
4. **Format** — Run `make format-fix` to ensure consistent code style.
5. **Build** — Run `make reldebug` to compile the code.
6. **Test** — Run the test suite to ensure nothing is broken.
7. **Submit** — Open a pull request for code review.

---

## Key Concepts Glossary

| Term | Plain-English Meaning |
|------|-----------------------|
| **SQL** | A standard language for asking questions to a database ("Structured Query Language") |
| **AST** | Abstract Syntax Tree — a tree-shaped data structure representing the structure of a SQL query |
| **Logical Plan** | A step-by-step description of *what* a query should do, without specifying *how* |
| **Physical Plan** | A concrete plan specifying exactly *how* operations will be executed on hardware |
| **Vectorized execution** | Processing thousands of rows at once instead of one at a time |
| **Column storage** | Storing all values of one column together, rather than storing rows together |
| **Buffer manager** | The part of DuckDB that decides what data to keep in memory vs. on disk |
| **MVCC** | Multi-Version Concurrency Control — a technique for letting multiple users read/write simultaneously without conflicts |
| **Extension** | An optional add-on module that adds new features to DuckDB |
| **Catalog** | DuckDB's internal directory of tables, columns, and functions |
| **Pipeline** | A chain of operations that data flows through during query execution |
| **WAL** | Write-Ahead Log — a crash-safety mechanism that records changes before applying them |
| **Predicate pushdown** | An optimization that filters data as early as possible to reduce work downstream |

---

## Summary

DuckDB is a complete, self-contained analytical database system. Its codebase is organized around the journey a query takes from text to results:

1. **You type SQL** → the **parser** reads it
2. The **planner** creates a logical plan
3. The **optimizer** makes the plan smarter
4. The **executor** runs the plan, in parallel, in batches
5. The **storage** layer provides data from disk

Everything else — extensions, tools for other languages, tests, utilities — supports this core pipeline.

The project is designed to be embedded (used inside another program) rather than run as a server, making it lightweight, easy to use, and extremely fast for the data analysis workloads it is built for.
