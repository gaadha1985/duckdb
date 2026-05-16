# DuckDB Parser — Explained Line by Line

This document walks through every part of DuckDB's SQL parser (`src/parser/`)
in plain language. No programming background is assumed.

---

## What Is a Parser?

Before reading any code, it helps to understand what a parser does.

When you type a sentence like:

> "Please bring me a large coffee with oat milk and no sugar."

A human brain automatically breaks it into:
- **Action:** bring
- **Who:** me
- **What:** coffee
- **Size:** large
- **Modifications:** oat milk, no sugar

A **parser** does the exact same thing for SQL. When you type:

```sql
SELECT name, age FROM people WHERE age > 30 ORDER BY age
```

The parser breaks it into:
- **Action:** SELECT
- **Columns to return:** `name`, `age`
- **Source table:** `people`
- **Filter:** `age > 30`
- **Sort order:** `age` ascending

The result is not text anymore — it is a tree-shaped data structure the rest of
DuckDB can work with programmatically. This tree is called an
**Abstract Syntax Tree**, or AST.

---

## The Big Map: How the Parser Is Organized

```
Your SQL text
     |
     v
[ parser.cpp ]                ← Entry point. Cleans up the text, calls PEG.
     |
     |---> [ peg/peg_parser.cpp ]     ← The actual grammar-matching engine.
     |         |
     |         |---> [ peg/tokenizer/ ]       ← Splits text into tokens (words).
     |         |
     |         └---> [ peg/grammar/*.gram ]   ← The grammar rules (what is valid SQL).
     |
     └---> [ peg/transformer/ ]   ← Converts grammar match into AST objects.
               |
               |---> [ statement/*.cpp ]        ← One object per SQL statement type.
               |---> [ query_node/*.cpp ]        ← Node types (SELECT, INSERT, etc.)
               |---> [ expression/*.cpp ]        ← Expressions (math, columns, functions).
               |---> [ tableref/*.cpp ]          ← Table references (FROM clause).
               |---> [ constraints/*.cpp ]       ← Column constraints (NOT NULL, etc.)
               └---> [ parsed_data/*.cpp ]       ← Metadata for CREATE/ALTER/DROP.
```

Every arrow means "calls" or "produces". Let's walk through each part
from the top down.

---

## 1. `parser.cpp` — The Front Door

**File size:** ~654 lines  
**What it does:** This is the first file that runs when DuckDB needs to parse
SQL. Think of it as a receptionist: it cleans up your query and then hands it
to the right specialist.

### Step 1 — Validate UTF-8

SQL is text, and text is stored as bytes. Some bytes are invalid — they don't
represent any real character. The parser's first job is to check that the input
is valid text.

If the SQL contains invalid bytes, an error is reported immediately before
doing anything else.

### Step 2 — Strip Unicode Spaces (`StripUnicodeSpaces`)

Humans copy-paste SQL from all kinds of places — PDFs, web pages, word
processors — and many of those sources insert invisible "fancy" space
characters instead of a regular space. For example:

- **Non-breaking space** (U+00A0) — looks like a space but is not.
- **Em space** (U+2003) — a wider space used in typography.
- **Zero-width space** (U+200B) — invisible but still there.
- **Byte Order Mark** (U+FEFF) — a character at the start of some files.

If DuckDB saw `SELECT·name·FROM·people` where `·` is a non-breaking space, it
would fail to parse even though the query looks perfectly fine visually.

`StripUnicodeSpaces` scans every character and replaces all of these exotic
spaces with a plain ASCII space. It is careful not to replace them inside
quoted strings (where the space is intentional content, not formatting).

It uses a small **state machine** — a set of modes it switches between:

| Mode | Meaning |
|------|---------|
| `regular` | Normal SQL outside any string |
| `in_quotes` | Inside a `'single-quoted'` string |
| `in_dollar_quotes` | Inside a `$$dollar-quoted$$` string |
| `in_comment` | Inside a `-- line comment` or `/* block comment */` |

In `regular` mode, exotic spaces are replaced. In all other modes, they are
left alone.

### Step 3 — Check for Extension Overrides

DuckDB supports **parser extensions** — plugins that can define their own SQL
syntax (new statement types, new keywords). Before the normal grammar runs,
the parser checks if any loaded extension wants to handle this SQL. If an
extension says "I recognize this!", it handles the parsing entirely and the
built-in grammar is skipped.

There are two override modes:
- `DEFAULT_OVERRIDE` — extension gets the first chance; falls back to built-in
  if it does not recognize the SQL.
- `STRICT_OVERRIDE` — extension gets full control; built-in grammar is not
  consulted at all.

### Step 4 — Run the PEG Parser

The cleaned SQL text is passed to `peg_parser.cpp`. This is where the heavy
lifting happens. The result is a list of **statement objects** — one for each
SQL statement in the input (you can run multiple statements separated by `;`).

### Step 5 — Post-processing

After parsing, the parser:
- Records where each statement starts and ends in the original text (character
  positions), so error messages can point to the right place.
- Stores the original SQL text with each statement (useful for `EXPLAIN` and
  query profiling).
- For `CREATE` statements (creating tables, views, functions), stores the full
  original SQL text — this is needed so DuckDB can re-show you the definition
  later with `SHOW CREATE TABLE`.

### Helper methods

`parser.cpp` also exposes a set of convenience methods that parse *pieces* of
SQL rather than full statements:

| Method | Parses |
|--------|--------|
| `ParseExpressionList` | A comma-separated list of expressions |
| `ParseGroupByList` | A GROUP BY clause |
| `ParseOrderList` | An ORDER BY clause |
| `ParseUpdateList` | The SET clause of an UPDATE |
| `ParseValuesList` | A VALUES(...) list |
| `ParseColumnList` | A list of column names |
| `ParseColumnDefinition` | A single column definition (name + type) |

These are used by tools like the query autocompletion engine which needs to
parse partial SQL.

---

## 2. `peg/` — The Grammar Engine

This folder contains the core technology that actually understands SQL grammar.
DuckDB uses a technique called **PEG (Parsing Expression Grammar)**.

### What is PEG?

A PEG is a set of rules written in a formal language that describes exactly what
valid SQL looks like. Think of it like a very precise recipe:

- "A SELECT statement starts with the word SELECT, followed by a column list,
  optionally followed by FROM and a table name, optionally followed by WHERE
  and a condition..."

PEG rules are unambiguous — each rule has one deterministic answer. When two
rules could both match, PEG always tries the first one. If the first one fails,
it tries the second. This is called **ordered choice**.

### `peg/grammar/statements/*.gram` — The Grammar Rules

DuckDB's grammar is written in 31+ `.gram` files, one per statement type:

| File | Describes |
|------|-----------|
| `select.gram` | SELECT statements |
| `insert.gram` | INSERT INTO statements |
| `update.gram` | UPDATE statements |
| `delete.gram` | DELETE statements |
| `create_table.gram` | CREATE TABLE statements |
| `create_view.gram` | CREATE VIEW statements |
| `expression.gram` | All expressions (math, comparisons, functions) |
| `common.gram` | Shared rules used everywhere |
| ...and 23 more | Other SQL commands |

These are not C++ code — they are written in a special grammar notation.
Here is what that notation looks like:

```
# A SELECT statement
SelectStatement <- 'SELECT' SelectList ('FROM' TableRef)? ('WHERE' Expr)?

# A comma-separated list of expressions
SelectList <- Expr (',' Expr)*

# An expression is either a column reference, a number, or a function call
Expr <- ColumnRef / NumberLiteral / FunctionCall
```

The symbols mean:
- `'SELECT'` — the literal word SELECT (case-insensitive).
- `A B` — A followed by B (both must match in order).
- `(A)?` — A is optional (zero or one times).
- `(A)*` — A can repeat (zero or more times).
- `(A)+` — A must appear at least once.
- `A / B` — try A first; if it fails, try B (ordered choice).
- `!A` — negative lookahead: match only if A does *not* appear here.

### `peg/grammar/keywords/` — The Keyword Lists

SQL has reserved words (`SELECT`, `WHERE`, `FROM`) that have special meaning
and cannot be used as column or table names. DuckDB organizes keywords into
five lists:

| File | Meaning |
|------|---------|
| `reserved_keyword.list` | Cannot be used as identifiers at all (e.g., `SELECT`, `WHERE`) |
| `unreserved_keyword.list` | Reserved but can be used as identifiers in most places |
| `column_name_keyword.list` | Allowed as column names |
| `func_name_keyword.list` | Allowed as function names |
| `type_name_keyword.list` | Allowed as type names (e.g., `INTEGER`, `TEXT`) |

These lists are read by a build script that generates C++ lookup tables
(`keyword_map.cpp`) — fast hash maps for checking "is this word a keyword?"

### `peg/tokenizer/` — Splitting Text into Words

Before the grammar rules run, the input text is split into **tokens** —
meaningful chunks. This is like splitting a sentence into individual words.

For example, `SELECT name FROM people WHERE age > 30` becomes:
```
[SELECT] [name] [FROM] [people] [WHERE] [age] [>] [30]
```

Each token has a **type** (keyword, identifier, number, operator, punctuation,
string literal, etc.) and a **position** in the original text.

Three tokenizer files:
- `base_tokenizer.cpp` — common tokenization logic shared by all tokenizers.
- `parser_tokenizer.cpp` — tokenizes SQL for the grammar matcher.
- `highlight_tokenizer.cpp` — tokenizes SQL for syntax highlighting in the shell
  (assigns colors to keywords, strings, operators, etc.).

### `peg/peg_parser.cpp` — Matching Grammar to Tokens

Once tokens exist, `peg_parser.cpp` runs the grammar matcher:

1. It loads the compiled grammar rules (from `inlined_grammar.hpp` — a
   generated file containing all `.gram` files merged into one).
2. It tries to match the token stream against the grammar rules.
3. If the match succeeds, it returns a **parse result tree** — a raw tree
   structure showing which grammar rules matched which tokens.
4. If the match fails, it produces an error message pointing to the token that
   could not be matched.

**Caching:** Grammar matching is expensive the first time (it compiles the
grammar). DuckDB caches the compiled grammar (`ParserCache`) so subsequent
queries in the same session are much faster.

### `peg/transformer/` — Converting Grammar Matches into AST Objects

A raw grammar match is still not useful — it just says "the first 3 tokens
matched the SELECT rule." The **transformer** converts this into real C++
objects that the planner can work with.

The transformer folder contains 47+ files, each responsible for one grammar
rule. The naming is consistent: `transform_select.cpp` handles SELECT,
`transform_insert.cpp` handles INSERT, and so on.

#### How a transformer works (step by step)

Imagine the grammar rule for a simple column reference:
```
ColumnRef <- (TableName '.')? ColumnName
```

The transformer for this rule:

1. Receives the raw parse result for `ColumnRef`.
2. Checks if the optional part `(TableName '.')?` was present.
3. If yes: extracts the table name string and column name string.
4. If no: extracts only the column name string.
5. Creates a `ColumnRefExpression` object with those strings.
6. Returns it.

Every transformer follows this pattern:
- **Cast** the raw parse result to the expected type
  (`ListParseResult`, `ChoiceParseResult`, `OptionalParseResult`, etc.).
- **Extract** children by index matching the grammar rule's order.
- **Recursively transform** child elements (e.g., transform expressions inside a
  WHERE clause).
- **Construct and return** the appropriate AST object.

#### `peg_transformer_factory.cpp` — The Registry

This file registers all 47+ transformers in a lookup table: grammar rule name →
transformer function. When the grammar matcher says "this matched rule
`SelectStatement`", the factory looks up and calls `TransformSelectStatement`.

#### `peg_transformer.cpp` — Shared Transformer State

Holds state shared across all transformers during one parse:
- **Named parameter tracking** — for queries like `SELECT $name`, keeps track
  of all named parameters and their positions.
- **Parameter type checking** — ensures you do not mix named parameters (`$name`)
  with positional parameters (`$1`) in the same query.
- **PIVOT helper** — PIVOT syntax can automatically generate `CREATE ENUM`
  statements; this method generates them.

---

## 3. `sql_statement.hpp` and `statement/*.cpp` — SQL Statements

Every SQL statement (SELECT, INSERT, CREATE TABLE, etc.) is represented as a
C++ object. They all inherit from a common base class `SQLStatement`.

### `sql_statement.hpp` — The Base Class

Every statement, regardless of type, has:

| Field | Meaning |
|-------|---------|
| `type` | What kind of statement (`SELECT`, `INSERT`, `CREATE`, etc.) |
| `stmt_location` | Where this statement starts in the original SQL text |
| `stmt_length` | How many characters long this statement is |
| `query` | The original SQL text of this statement |
| `named_param_map` | Map of parameter names to positions for parameterized queries |

Every statement must implement two methods:
- `ToString()` — convert back to a SQL string (used by EXPLAIN and debugging).
- `Copy()` — create a deep copy of the statement.

### `statement/select_statement.cpp` — SELECT

`SelectStatement` is the simplest statement wrapper. It contains one thing:

- `node` — a `QueryNode` object (almost always a `SelectNode`) that holds the
  actual content of the SELECT query.

Why the extra layer? Because a SELECT can be complex — it can contain UNION,
INTERSECT, EXCEPT, or CTEs. The `QueryNode` abstraction handles all of that.

Methods:
- `Copy()` — creates a deep copy of the statement and its node.
- `Equals()` — compares two SELECT statements for equality.
- `ToString()` — delegates to `node->ToString()`.

### Other statement files

Each file follows the same pattern. Here is what each statement type holds:

| Statement | Key contents |
|-----------|-------------|
| `insert_statement.cpp` | Target table, columns list, values or SELECT subquery |
| `update_statement.cpp` | Target table, SET assignments, WHERE clause |
| `delete_statement.cpp` | Target table, WHERE clause, USING clause |
| `create_statement.cpp` | `CreateInfo` object (see `parsed_data/`) |
| `alter_statement.cpp` | `AlterInfo` object |
| `drop_statement.cpp` | What to drop (table, view, index), name, IF EXISTS flag |
| `copy_statement.cpp` | Source/destination, format (CSV, Parquet), options |
| `transaction_statement.cpp` | BEGIN, COMMIT, or ROLLBACK |
| `explain_statement.cpp` | The inner statement to explain, format |
| `prepare_statement.cpp` | Statement name, the SQL to prepare |
| `execute_statement.cpp` | Prepared statement name, parameter values |
| `pragma_statement.cpp` | Pragma name and optional value |
| `vacuum_statement.cpp` | Which table to vacuum (or whole database) |
| `merge_into_statement.cpp` | Target, source, WHEN MATCHED / NOT MATCHED clauses |
| `attach_statement.cpp` | File path, database name, options |
| `set_statement.cpp` | Variable name, new value, scope (LOCAL or GLOBAL) |

---

## 4. `query_node/*.cpp` — Query Node Types

A `QueryNode` represents the *body* of a query — what data to produce and how.
Different SQL constructs produce different node types.

### `query_node/select_node.cpp` — The SELECT Node

This is the most important query node. It holds every clause of a SELECT:

| Field | SQL Clause | Example |
|-------|-----------|---------|
| `select_list` | SELECT ... | `name, age, salary * 1.1` |
| `from_table` | FROM ... | `orders JOIN customers ON ...` |
| `where_clause` | WHERE ... | `age > 30 AND city = 'NYC'` |
| `groups` | GROUP BY ... | `region, year` |
| `having` | HAVING ... | `SUM(sales) > 10000` |
| `qualify` | QUALIFY ... | `RANK() OVER (...) = 1` |
| `sample` | USING SAMPLE ... | `10%` |
| `modifiers` | ORDER BY, LIMIT, OFFSET | `ORDER BY age DESC LIMIT 10` |
| `cte_map` | WITH ... AS ... | `WITH totals AS (SELECT ...)` |
| `aggregate_handling` | (internal) | Whether aggregates are allowed |

`ToString()` assembles all these pieces back into a SQL string — useful for
displaying query plans.

### `query_node/set_operation_node.cpp` — UNION, INTERSECT, EXCEPT

When you combine two queries with `UNION ALL` or `INTERSECT`, the result is a
`SetOperationNode`. It holds:
- A **left** query node (first SELECT).
- A **right** query node (second SELECT).
- The **operation type** (UNION, UNION ALL, INTERSECT, EXCEPT).

These can be nested: `A UNION B EXCEPT C` becomes a tree of SetOperationNodes.

### `query_node/cte_node.cpp` — WITH ... AS (Common Table Expressions)

`WITH totals AS (SELECT region, SUM(sales) FROM orders GROUP BY region)`

A CTE node holds:
- The **name** of the CTE (`totals`).
- The **query** inside the CTE (a full SelectNode).
- Whether it is **RECURSIVE**.

CTEs are stored in the `cte_map` of the outer SelectNode and can be referenced
by name in the FROM clause.

### `query_node/recursive_cte_node.cpp` — RECURSIVE CTEs

A recursive CTE calls itself repeatedly (like a loop). It holds:
- A **base case** query (the starting point).
- A **recursive case** query (references the CTE's own name).
- The CTE's name.

### `query_node/insert_query_node.cpp` and `update_query_node.cpp`

INSERT and UPDATE statements can have complex FROM clauses and CTEs, so they
use their own query nodes with the relevant fields.

---

## 5. `parsed_expression.hpp` and `expression/*.cpp` — Expressions

An **expression** is any piece of SQL that produces a value. Everything from a
simple column name to a complex nested function call is an expression.

All expressions inherit from `ParsedExpression`.

### `parsed_expression.hpp` — The Base Class

Every expression has:

| Method | What it checks |
|--------|---------------|
| `IsAggregate()` | Does this contain `SUM()`, `COUNT()`, etc.? |
| `IsWindow()` | Does this contain a window function (`OVER`)? |
| `HasSubquery()` | Does this contain a `(SELECT ...)` inside? |
| `IsScalar()` | Is this a pure computation with no table references? |
| `HasParameter()` | Does this contain `$1` or `$name`? |

These are **recursive** — they walk the whole expression tree. For example,
`IsAggregate()` returns true for `SUM(price * 1.1)` because somewhere inside
there is an aggregate function (`SUM`).

Every expression also implements:
- `Equals()` — deep equality comparison.
- `Hash()` — for use in hash maps.
- `Copy()` — deep copy.
- `ToString()` — converts back to SQL text.

### `expression/constant_expression.cpp` — Literal Values

A literal value in SQL: `42`, `'hello'`, `3.14`, `TRUE`, `NULL`, `DATE '2024-01-01'`.

Holds:
- `value` — the actual value with its type (integer, string, float, date, etc.).

### `expression/columnref_expression.cpp` — Column References

Any time a column name appears in SQL: `age`, `orders.price`,
`main.orders.price`, `mydb.main.orders.price`.

The column name is stored as a **vector of name parts** (1 to 4 parts):
- 1 part: `age` → just a column name.
- 2 parts: `orders.price` → table + column.
- 3 parts: `main.orders.price` → schema + table + column.
- 4 parts: `mydb.main.orders.price` → catalog + schema + table + column.

Key methods:
- `GetColumnName()` — returns the last part (always the column name).
- `GetTableName()` — returns the second-to-last part (the table name).
- `IsQualified()` — returns true if there is more than just a column name.
- `Equal(a, b)` — case-insensitive comparison (SQL is case-insensitive for names).

### `expression/function_expression.cpp` — Function Calls

Any function call: `SUM(price)`, `UPPER(name)`, `date_diff('day', start, end)`,
`COUNT(DISTINCT customer_id)`.

Holds:
- `function_name` — the function's name.
- `catalog`, `schema` — optional fully-qualified path.
- `children` — the arguments (each is itself an expression).
- `distinct` — true if `COUNT(DISTINCT ...)` syntax was used.
- `filter` — for aggregate filter syntax: `SUM(price) FILTER (WHERE status = 'paid')`.
- `order_bys` — for ordered aggregates: `STRING_AGG(name ORDER BY age)`.
- `is_operator` — true when written as `a + b` (which is actually `+(a, b)`).

### `expression/comparison_expression.cpp` — Comparisons

Any comparison: `age > 30`, `name = 'Alice'`, `price <> 0`, `qty <= 100`.

Holds:
- `left` — left side expression.
- `right` — right side expression.
- `type` — the comparison operator (EQUAL, NOT_EQUAL, LESS_THAN, etc.).

### `expression/conjunction_expression.cpp` — AND / OR

Logical combinations: `age > 30 AND city = 'NYC'`, `status = 'A' OR status = 'B'`.

Holds:
- `children` — list of expressions being combined.
- `type` — AND or OR.

### `expression/case_expression.cpp` — CASE WHEN

```sql
CASE WHEN age < 18 THEN 'minor'
     WHEN age < 65 THEN 'adult'
     ELSE 'senior'
END
```

Holds:
- `case_checks` — list of (WHEN condition, THEN result) pairs.
- `else_expr` — the ELSE value (optional).
- `root_expression` — the value being switched on for simple CASE syntax.

### `expression/cast_expression.cpp` — Type Conversion

`CAST(price AS INTEGER)`, `price::INTEGER`.

Holds:
- `child` — the expression being converted.
- `cast_type` — the target data type.
- `try_cast` — true for `TRY_CAST(...)` which returns NULL on failure instead
  of an error.

### `expression/between_expression.cpp` — BETWEEN

`age BETWEEN 18 AND 65` (equivalent to `age >= 18 AND age <= 65`).

Holds:
- `input` — the value being tested (`age`).
- `lower` — lower bound (`18`).
- `upper` — upper bound (`65`).

### `expression/subquery_expression.cpp` — Subqueries Inside Expressions

```sql
SELECT name FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees)
```

The `(SELECT AVG(salary) FROM employees)` part is a subquery expression. Holds:
- `subquery` — a full `SelectStatement` nested inside.
- `subquery_type` — SCALAR (returns one value), EXISTS, NOT EXISTS, IN, NOT IN, ANY, ALL.
- `comparison_type` — for ANY/ALL, which comparison to use.

### `expression/window_expression.cpp` — Window Functions

```sql
RANK() OVER (PARTITION BY region ORDER BY sales DESC)
```

Holds:
- `function_name` — the window function name (`RANK`, `SUM`, `LAG`, etc.).
- `children` — arguments to the function.
- `partitions` — PARTITION BY expressions.
- `orders` — ORDER BY expressions.
- `start` / `end` — window frame boundaries (ROWS BETWEEN ...).
- `ignore_nulls` — true for `IGNORE NULLS` syntax.
- `filter_expr` — filter clause.

### `expression/star_expression.cpp` — SELECT *

The `*` in `SELECT *` or `table.*`.

Holds:
- `relation_name` — optional table name for `orders.*` syntax.
- `exclude_list` — columns to exclude: `SELECT * EXCLUDE (password)`.
- `replace_list` — columns to replace: `SELECT * REPLACE (price * 1.1 AS price)`.

### `expression/lambda_expression.cpp` — Lambda Functions

Used with list operations: `list_transform([1,2,3], x -> x * 2)`.

Holds:
- `lhs` — the parameter(s) (can be a single name or a list).
- `expr` — the body expression (`x * 2`).

### `expression/parameter_expression.cpp` — Parameterized Queries

Placeholders in prepared statements: `$1`, `$2`, `$name`.

Holds:
- `identifier` — the parameter name or number.
- `is_named` — true for `$name`, false for `$1`.

### `expression/collate_expression.cpp` — Text Collation

`name COLLATE NOCASE` — compare text using a specific sorting/comparison rule.

Holds:
- `child` — the expression being collated.
- `collation` — the collation name.

### `expression/operator_expression.cpp` — Operators

Unary and binary operators that are not comparisons:
- `NOT condition`
- `value IS NULL`
- `value IS NOT NULL`
- `value IN (list)`
- Arithmetic: already handled as function calls, but some operators have their own nodes.

---

## 6. `tableref/*.cpp` — Table References (FROM Clause)

A **table reference** is anything that appears in a FROM clause — a table name,
a subquery, a join, a function call, etc.

All table references inherit from `TableRef`.

### `tableref/basetableref.cpp` — A Plain Table Name

`FROM orders` or `FROM main.orders` or `FROM mydb.main.orders`.

Holds:
- `catalog_name`, `schema_name`, `table_name` — the qualified table name.
- `alias` — optional alias: `FROM orders o`.
- `column_name_alias` — for renaming columns: `FROM orders AS o(id, price, qty)`.
- `at_clause` — temporal versioning: `FROM orders AT (VERSION = 5)`.
- `sample` — random sampling: `FROM orders TABLESAMPLE (1%)`.

### `tableref/joinref.cpp` — JOIN

```sql
orders JOIN customers ON orders.customer_id = customers.id
```

Holds:
- `left` — left table reference.
- `right` — right table reference.
- `condition` — the ON clause expression.
- `using_columns` — for `JOIN ... USING (column_name)` syntax.
- `join_type` — INNER, LEFT OUTER, RIGHT OUTER, FULL OUTER, CROSS, SEMI, ANTI.
- `ref_type` — REGULAR, NATURAL, CROSS, POSITIONAL, ASOF.

### `tableref/subqueryref.cpp` — Subquery in FROM

```sql
FROM (SELECT region, SUM(sales) AS total FROM orders GROUP BY region) AS totals
```

Holds:
- `subquery` — the full SelectStatement inside the parentheses.
- `alias` — the name given to the subquery result (`totals`).
- `column_name_alias` — optional column name overrides.

### `tableref/table_function.cpp` — Table-Valued Functions

```sql
FROM read_csv('data.csv')
FROM range(1, 100)
FROM generate_series(0, 10, 2)
```

Holds:
- `function` — a `FunctionExpression` with the function name and arguments.
- `alias` — optional alias.
- `column_name_alias` — optional column renaming.

### `tableref/expressionlistref.cpp` — VALUES Clause

```sql
FROM (VALUES (1, 'Alice'), (2, 'Bob'), (3, 'Carol')) AS t(id, name)
```

Holds:
- `values` — list of rows, each row being a list of expressions.
- `alias` — name for the result.
- `expected_names` — the column names.

### `tableref/pivotref.cpp` — PIVOT

```sql
FROM orders PIVOT (SUM(sales) FOR region IN ('East', 'West', 'North'))
```

Holds:
- `source` — the underlying table reference.
- `aggregates` — aggregate functions to compute.
- `pivots` — the pivot column and its values.
- `alias` — result alias.

### `tableref/emptytableref.cpp` — No Table

`SELECT 1 + 1` has no FROM clause. This placeholder represents "no source table."

### `tableref/showref.cpp` — SHOW Statements

`SHOW TABLES`, `SHOW DATABASES`, `SHOW ALL TABLES`.

Holds the type of thing to show and any filter conditions.

### `tableref/at_clause.cpp` — Time Travel

`FROM orders AT (VERSION = 42)` or `FROM orders AT (TIMESTAMP = '2024-01-01')`.

Used for reading historical versions of a table. Holds the version type and value.

---

## 7. `constraints/*.cpp` — Column Constraints

Constraints are rules applied to columns or tables. They are parsed and stored
as objects, then enforced during INSERT/UPDATE.

### `constraints/not_null_constraint.cpp`

`NOT NULL` — the column must always have a value.

Holds:
- `index` — which column this constraint applies to.

### `constraints/unique_constraint.cpp`

`UNIQUE` or `PRIMARY KEY` — values must be unique across all rows.

Holds:
- `columns` — list of column names.
- `is_primary_key` — true for PRIMARY KEY, false for UNIQUE.

### `constraints/check_constraint.cpp`

`CHECK (age >= 0)` — a custom rule that must always be true.

Holds:
- `expression` — the boolean expression to check.

### `constraints/foreign_key_constraint.cpp`

`FOREIGN KEY (customer_id) REFERENCES customers(id)` — links to another table.

Holds:
- `fk_columns` — the columns in this table.
- `info` — reference to the target table and its columns.

---

## 8. `parsed_data/*.cpp` — Metadata for DDL Statements

DDL statements (Data Definition Language — CREATE, ALTER, DROP) carry a lot of
metadata. Each has its own `Info` class.

### `parsed_data/create_table_info.cpp`

```sql
CREATE TABLE orders (
    id INTEGER PRIMARY KEY,
    customer_id INTEGER NOT NULL,
    price DECIMAL(10,2),
    created_at TIMESTAMP DEFAULT NOW()
)
```

Holds:
- `table` — table name.
- `schema`, `catalog` — optional namespace.
- `columns` — list of `ColumnDefinition` objects (name, type, default, constraints).
- `constraints` — table-level constraints.
- `query` — if `CREATE TABLE ... AS SELECT ...`, the SELECT statement.
- `on_conflict` — what to do if table already exists (ERROR, IGNORE, REPLACE).

### `parsed_data/create_view_info.cpp`

```sql
CREATE VIEW active_customers AS
SELECT * FROM customers WHERE status = 'active'
```

Holds:
- `view_name`, `schema`, `catalog`.
- `query` — the SELECT statement behind the view.
- `aliases` — optional column name overrides.

### `parsed_data/alter_table_info.cpp`

```sql
ALTER TABLE orders ADD COLUMN status VARCHAR DEFAULT 'pending'
ALTER TABLE orders RENAME COLUMN price TO amount
ALTER TABLE orders DROP COLUMN notes
```

Holds:
- `table` — which table to alter.
- `type` — what kind of alteration (ADD_COLUMN, RENAME_COLUMN, DROP_COLUMN, etc.).
- The new column definition, or the old/new name, depending on type.

### `parsed_data/drop_info.cpp`

```sql
DROP TABLE IF EXISTS orders
DROP VIEW active_customers CASCADE
```

Holds:
- `name` — what to drop.
- `type` — TABLE, VIEW, INDEX, FUNCTION, etc.
- `if_exists` — whether to silently ignore if it does not exist.
- `cascade` — whether to also drop dependent objects.

### `parsed_data/copy_info.cpp`

```sql
COPY orders FROM 'data.csv' (FORMAT CSV, HEADER true)
COPY orders TO 'output.parquet' (FORMAT PARQUET)
```

Holds:
- `table` — target table.
- `file_path` — source or destination file.
- `format` — CSV, Parquet, JSON, etc.
- `options` — key-value pairs of format-specific options.
- `is_from` — true for reading (FROM), false for writing (TO).

### `parsed_data/sample_options.cpp`

```sql
FROM orders TABLESAMPLE BERNOULLI (10 PERCENT)
FROM orders USING SAMPLE 1000 ROWS
```

Holds:
- `sample_size` — the amount to sample.
- `method` — BERNOULLI (probabilistic) or SYSTEM (block-based) or RESERVOIR.
- `seed` — optional random seed for reproducibility.

---

## 9. End-to-End Example: Parsing a SELECT Query

Let's trace exactly what happens when you send this SQL to DuckDB:

```sql
SELECT name, SUM(price) AS total
FROM orders
WHERE status = 'paid'
GROUP BY name
ORDER BY total DESC
```

### Step 1 — `parser.cpp` receives the SQL string

The `ParseQuery` method is called with the full SQL text.

### Step 2 — UTF-8 validation

Checks every byte. All fine here — standard ASCII.

### Step 3 — `StripUnicodeSpaces`

No fancy spaces found. The string passes through unchanged.

### Step 4 — Extension override check

No extension is registered. Continue with built-in grammar.

### Step 5 — Tokenization (`parser_tokenizer.cpp`)

The text is split into tokens:

```
[SELECT] [name] [,] [SUM] [(] [price] [)] [AS] [total]
[FROM] [orders]
[WHERE] [status] [=] ['paid']
[GROUP] [BY] [name]
[ORDER] [BY] [total] [DESC]
```

Each token has a type (keyword, identifier, operator, string literal) and a
position in the original text.

### Step 6 — PEG matching (`peg_parser.cpp`)

The grammar rule for `SelectStatement` is applied to the token stream. It
matches successfully, producing a raw parse result tree like:

```
SelectStatement
  ├── select_list
  │     ├── ColumnRef: "name"
  │     └── FunctionCall: SUM(ColumnRef: "price") AS "total"
  ├── from_clause: TableName: "orders"
  ├── where_clause: Comparison(ColumnRef: "status" = StringLiteral: "paid")
  ├── group_by: ColumnRef: "name"
  └── order_by: ColumnRef: "total" DESC
```

### Step 7 — Transformation (`peg/transformer/`)

The transformer factory finds `TransformSelectStatement` and calls it.

It calls child transformers for each clause:

- `TransformSelectList` → creates two `ParsedExpression` objects:
  - A `ColumnRefExpression` with `column_names = ["name"]`.
  - A `FunctionExpression` with `function_name = "sum"`,
    `children = [ColumnRefExpression("price")]`, and alias `"total"`.

- `TransformFromClause` → creates a `BaseTableRef` with `table_name = "orders"`.

- `TransformWhereClause` → creates a `ComparisonExpression`:
  - `left = ColumnRefExpression("status")`
  - `right = ConstantExpression("paid")`
  - `type = EQUAL`

- `TransformGroupBy` → creates a list with `ColumnRefExpression("name")`.

- `TransformOrderBy` → creates an order modifier with
  `ColumnRefExpression("total")` and `DESCENDING`.

### Step 8 — Final object assembly

A `SelectNode` is created with all the above fields populated. It is wrapped
in a `SelectStatement`. The statement is added to `parser.statements`.

### Step 9 — Post-processing

The parser records:
- The statement starts at character 0.
- The statement is 87 characters long.
- The original SQL text is stored in `query`.

### Step 10 — Result

The caller receives a `vector<unique_ptr<SQLStatement>>` with one
`SelectStatement` inside. The planner picks this up next.

---

## 10. Key Architectural Patterns

### Ordered choice is deterministic

PEG never backtracks randomly. When parsing `age > 30`, the grammar has rules
for `>`, `>=`, `<>`, etc. PEG tries them in the order they are listed. If `>=`
is listed before `>`, it will never accidentally match `>` when `>=` is meant.
The grammar author controls this by ordering rules carefully.

### The visitor pattern

`ParsedExpressionIterator` can walk any expression tree. Methods like
`IsAggregate()` use it to check every node without writing recursive code
everywhere. Any transformation (like replacing one expression with another) can
also be expressed as an iterator visit.

### Deep copying everywhere

Every expression and statement implements `Copy()`. This is critical because:
- The same prepared statement can be re-used multiple times.
- The optimizer may create multiple variants of a query to compare.
- The planner may modify a copy without affecting the original.

### Everything can convert back to SQL

Every object implements `ToString()` — it can write itself back as valid SQL.
This is used for:
- `EXPLAIN` output.
- Showing view definitions (`SHOW CREATE VIEW`).
- Debugging the optimizer (showing intermediate query plans as SQL).

### Case-insensitive name comparison

SQL names are case-insensitive: `Orders`, `ORDERS`, and `orders` all refer to
the same table. All `Equals()` and `Hash()` methods on names use
case-insensitive comparison to respect this rule.

---

## Summary Table

| File / Folder | Role |
|--------------|------|
| `parser.cpp` | Entry point, UTF-8 validation, unicode cleanup, extension hooks |
| `peg/grammar/*.gram` | The formal grammar rules describing valid SQL |
| `peg/grammar/keywords/` | Lists of reserved and unreserved words |
| `peg/tokenizer/` | Splits raw text into typed tokens (words, numbers, operators) |
| `peg/peg_parser.cpp` | Matches tokens against grammar, produces raw parse tree |
| `peg/transformer/` | Converts raw parse tree into typed C++ AST objects |
| `sql_statement.hpp` | Base class for all statement types |
| `statement/*.cpp` | One class per SQL statement (SELECT, INSERT, CREATE, etc.) |
| `query_node/*.cpp` | Query body types (SELECT node, UNION node, CTE node, etc.) |
| `parsed_expression.hpp` | Base class for all expression types |
| `expression/*.cpp` | One class per expression type (column ref, function, comparison, etc.) |
| `tableref/*.cpp` | Table reference types (table name, join, subquery, function, values) |
| `constraints/*.cpp` | Column/table constraints (NOT NULL, UNIQUE, CHECK, FOREIGN KEY) |
| `parsed_data/*.cpp` | Metadata objects for DDL statements (CREATE, ALTER, DROP) |
