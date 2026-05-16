# DuckDB Planner — Explained Line by Line

This document walks through every part of DuckDB's planner (`src/planner/`)
in plain language. No programming background is assumed.

---

## What Is a Planner?

After the parser has turned your SQL text into a tree of objects (the AST), the
**planner** takes over. Its job has two parts:

1. **Binding** — connect every name in the query to the real thing it refers to.
   - `orders` → the actual `orders` table in the database.
   - `price` → the 4th column of the `orders` table, type DECIMAL.
   - `SUM` → the built-in aggregate function named SUM.
   - `$1` → the first query parameter.

2. **Logical planning** — build a tree of operations (called a **logical plan**)
   that describes *what* the query needs to do, step by step.

The logical plan is not yet concerned with speed or hardware. That is the
optimizer's job. The planner just produces a correct, structured description of
the computation.

### A library analogy

Think of the parser as someone who reads a shopping list and understands the
words. The planner is the person who takes that list to the store and confirms:
- "Apples" → aisle 3, produce section, Granny Smith variety.
- "Milk" → refrigerated section, 2% fat.
- They also lay out a route through the store (the plan).

---

## The Big Map: How the Planner Is Organized

```
Parsed SQL (AST from parser)
         |
         v
[ planner.cpp ]             ← Top-level coordinator
         |
         v
[ binder.cpp ]              ← Resolves all names to real catalog objects
         |
         |---> [ binder/statement/*.cpp ]    ← One binder per statement type
         |---> [ binder/query_node/*.cpp ]   ← Binds SELECT, UNION, CTEs
         |---> [ binder/tableref/*.cpp ]     ← Binds FROM clause
         |---> [ binder/expression/*.cpp ]   ← Binds expressions
         |---> [ expression_binder/*.cpp ]   ← Clause-specific expression binding
         |
         v
[ Bound objects ]           ← AST nodes with full type and catalog info attached
         |
         v
[ binder/query_node/plan_*.cpp ]  ← Converts bound objects to LogicalOperators
         |
         v
[ operator/*.cpp ]          ← The logical operator tree (the plan)
         |
         v
[ subquery/flatten_dependent_join.cpp ]  ← Decorrelates subqueries
         |
         v
Final LogicalOperator tree  ← Ready for the optimizer
```

---

## 1. `planner.cpp` — The Top-Level Coordinator

**File size:** ~227 lines  
**What it does:** This is the entry point. When the executor needs a plan for a
SQL statement, it calls `Planner::CreatePlan()` here.

### The pipeline inside `CreatePlan()`

**Step 1 — Create a Binder**  
A new `Binder` object is created. The binder is the main workhorse; the planner
is just its driver.

**Step 2 — Bind the statement**  
`binder->Bind(statement)` is called. This takes the parsed AST and resolves
every name, type, and function reference against the catalog. The result is a
`BoundStatement` — a version of the AST where everything is confirmed to exist
and every type is known.

**Step 3 — Run post-bind extensions**  
Some DuckDB extensions want to inspect or modify the plan after binding but
before optimization. This hook runs them.

**Step 4 — Verify the plan (debug mode)**  
In debug builds, DuckDB serializes the plan to binary and deserializes it back,
checking that nothing was lost. This catches bugs in serialization logic early.

**Step 5 — Decorrelate subqueries**  
`FlattenDependentJoins::DecorrelateIndependent()` is called. This rewrites any
**correlated subqueries** (subqueries that reference columns from the outer
query) into equivalent joins. More on this below.

**Step 6 — Verify column bindings**  
`ColumnBindingResolver::Verify()` walks the entire logical plan and checks that
every column reference points to a real column that actually exists in the plan.
If anything is dangling or broken, an error is raised now rather than during
execution.

**Step 7 — Return**  
The completed `LogicalOperator` tree is stored in `planner.plan`. The planner
also records the output column names and types in `planner.names` and
`planner.types`.

### Key members of the Planner

| Member | Meaning |
|--------|---------|
| `binder` | The Binder object that does the actual binding work |
| `plan` | The resulting logical plan tree |
| `names` | Output column names for the final result |
| `types` | Output column types |
| `properties` | Metadata: is this a read or write? Does it return rows? |

---

## 2. `binder.cpp` — The Name Resolver

**File size:** ~649 lines  
**What it does:** The `Binder` class is the core of the planner. It visits every
node in the parsed AST and replaces every unresolved name with a confirmed,
typed reference to the real object.

### What "binding" means, concretely

Before binding:
```
ColumnRefExpression { column_names = ["price"] }
```
(We know the name "price" but nothing else.)

After binding:
```
BoundColumnRefExpression {
    return_type = DECIMAL(10, 2),
    binding = ColumnBinding(table_index=0, column_index=3)
}
```
(We know the exact column: table 0, column 3, type DECIMAL(10,2).)

### Binder hierarchy (scopes)

SQL has **scopes** — names visible in one part of a query may not be visible in
another. For example, a column alias defined in the SELECT list is not visible
in the WHERE clause (in standard SQL).

DuckDB handles scopes by creating a **chain of Binders** — each nested query
(subquery, CTE, view) gets its own child Binder that has a `parent` pointer to
the outer one. Column lookups walk up this chain.

```
Outer query binder (parent=null)
    └── Subquery binder (parent = outer)
            └── CTE binder (parent = subquery)
```

When a correlated subquery references a column from the outer query, the column
lookup walks up through parent binders until it finds the column. Each level it
crosses increments a **depth counter** stored in the `BoundColumnRefExpression`.
Depth > 0 means the column is **correlated** (belongs to an outer scope).

### Key members of the Binder

| Member | Meaning |
|--------|---------|
| `context` | Connection to the database catalog, settings, and session |
| `bind_context` | Tracks which tables and columns are visible in the current scope |
| `parent` | Parent binder for nested scopes (subqueries, CTEs) |
| `correlated_columns` | List of columns this scope borrows from an outer scope |
| `bound_views` | Tracks which views have been expanded to prevent infinite loops |
| `binder_type` | REGULAR (normal query) or VIEW (inside a view definition) |
| `global_binder_state` | Shared state across the entire binder chain (e.g., parameter count) |

### The main `Bind()` dispatch

`Binder::Bind(SQLStatement &statement)` looks at the statement type and calls
the appropriate specialized binder:

| Statement type | Calls |
|---------------|-------|
| SELECT | `bind_select.cpp` |
| INSERT | `bind_insert.cpp` |
| UPDATE | `bind_update.cpp` |
| DELETE | `bind_delete.cpp` |
| CREATE | `bind_create.cpp` → `bind_create_table.cpp` etc. |
| COPY | `bind_copy.cpp` |
| EXPLAIN | `bind_explain.cpp` |
| MERGE INTO | `bind_merge_into.cpp` |
| PRAGMA | `bind_pragma.cpp` |
| ...and 20 more | Various `bind_*.cpp` files |

---

## 3. `bind_context.cpp` — The Scope's Table of Contents

**File size:** ~30 KB  
**What it does:** For any given scope (the outer query, a subquery, etc.),
`BindContext` is the lookup table of everything visible: which tables are in
scope, which columns each table has, and which aliases exist.

### How column lookups work

When the binder sees `price` in an expression, it calls
`bind_context.GetMatchingBinding("price")`. The bind context:

1. Checks all registered table bindings to find which table has a column named
   `price`.
2. If exactly one table has it → resolves to that table's `price` column.
3. If multiple tables have it → **ambiguity error**: "column `price` is
   ambiguous; specify the table."
4. If none have it → tries the parent binder's context (correlated column).
5. If not found anywhere → error: "column `price` not found."

### What gets registered in the bind context

Every time a table reference is bound, its columns are registered:

- `FROM orders` → registers table `orders` with all its columns.
- `FROM orders AS o` → registers `o` as an alias for `orders`.
- `FROM (SELECT ...) AS sub` → registers `sub` with the subquery's output columns.
- `FROM generate_series(1, 10) AS gs(n)` → registers `gs` with column `n`.
- After `JOIN customers ON ...` → both `orders` and `customers` are visible.

---

## 4. Statement Binders (`binder/statement/`) — One Per SQL Command

Each file handles one type of SQL statement. They all follow the same pattern:
validate the statement, bind its sub-parts, and produce a `BoundStatement`
containing a `LogicalOperator` tree.

### `bind_select.cpp` — SELECT Statements

The simplest statement binder. A SELECT is almost entirely composed of its
query node (the SELECT body). This file:
1. Calls `binder.Bind(*statement.node)` to bind the query node.
2. Applies any result modifiers (ORDER BY, LIMIT, OFFSET) from the statement
   level.
3. Creates the logical plan from the bound node.
4. Wraps it in a `BoundStatement` with output names and types.

The real work is in `binder/query_node/bind_select_node.cpp`.

### `bind_insert.cpp` (~25 KB) — INSERT Statements

Much more complex. Handles:
- Identifying the target table in the catalog.
- Matching listed columns (`INSERT INTO t(a, b, c)`) to the table's actual columns.
- Binding the VALUES list or the SELECT subquery.
- Type-checking and coercing each value to the column's type (e.g., inserting
  an integer where a decimal is expected).
- Handling `ON CONFLICT DO NOTHING` / `ON CONFLICT DO UPDATE` (upsert logic).
- Handling the `RETURNING` clause (which columns to return after insert).
- Binding generated columns (columns whose value is computed automatically).

### `bind_create_table.cpp` (~29 KB) — CREATE TABLE

Handles:
- Validating the table name (does it already exist? IF NOT EXISTS?).
- Binding each column definition: name, type, default expression, constraints.
- Binding CHECK constraints (e.g., `CHECK (age >= 0)`) as expressions.
- Binding FOREIGN KEY references (verifying the referenced table/column exist).
- Handling `CREATE TABLE ... AS SELECT ...` by binding the SELECT subquery.
- Setting storage parameters (compression, row group size, etc.).

### `bind_copy.cpp` (~28 KB) — COPY FROM/TO

Handles:
- Identifying the source/destination file or table.
- Detecting or validating the file format (CSV, Parquet, JSON, etc.).
- Binding format-specific options (delimiter, header, null string, etc.).
- For `COPY FROM`: binding the target table and validating column compatibility.
- For `COPY TO`: binding the source query or table.

### `bind_merge_into.cpp` (~20 KB) — MERGE INTO

The most complex DML statement. Handles:
- Binding the target table and source table/query.
- Binding the join condition (`ON orders.id = updates.id`).
- For each `WHEN MATCHED THEN UPDATE SET ...` clause: binding the update
  expressions.
- For each `WHEN NOT MATCHED THEN INSERT ...` clause: binding the insert values.
- Ensuring no conflicting WHEN clauses.

### Other statement binders

| File | Key job |
|------|---------|
| `bind_update.cpp` | Validates target table, binds SET assignments and WHERE clause |
| `bind_delete.cpp` | Validates target table, binds WHERE and USING clauses |
| `bind_explain.cpp` | Binds the inner statement, wraps in EXPLAIN operator |
| `bind_pragma.cpp` | Looks up the pragma by name, validates arguments |
| `bind_drop.cpp` | Finds the object to drop, validates it exists |
| `bind_alter.cpp` | Validates the table to alter and the alteration |
| `bind_attach.cpp` | Validates file path and options for ATTACH DATABASE |
| `bind_export.cpp` | Binds the schema to export and destination path |
| `bind_load.cpp` | Validates and loads an extension |

---

## 5. Query Node Binding (`binder/query_node/`) — The SELECT Body

### `bind_select_node.cpp` (~29 KB) — The Most Important File

This file binds the body of a SELECT statement — all the clauses. It is called
by `bind_select.cpp` and produces a `BoundSelectNode`.

The binding order is carefully chosen because clauses depend on each other.
Here is the exact order and why:

**1. Bind the FROM clause**  
The table references must be resolved first because all column lookups in other
clauses depend on knowing which tables are in scope.

```sql
FROM orders JOIN customers ON orders.customer_id = customers.id
```
→ Registers `orders` and `customers` (and all their columns) in the bind context.

**2. Bind the SELECT list (partially — just collect aliases)**  
Column aliases defined in SELECT (`price * 1.1 AS adjusted_price`) can be
referenced in GROUP BY and ORDER BY. The aliases are collected first without
fully binding the expressions.

**3. Bind the WHERE clause**  
Now that tables are registered, column references in WHERE can be resolved.
Aggregate functions are not allowed in WHERE — the binder checks for this and
raises an error.

```sql
WHERE orders.status = 'paid' AND customers.region = 'East'
```
→ `orders.status` → table 0, column 2, type VARCHAR.  
→ `customers.region` → table 1, column 5, type VARCHAR.

**4. Bind GROUP BY**  
GROUP BY can reference:
- Column names from tables: `GROUP BY region`
- Column aliases from SELECT: `GROUP BY adjusted_price`
- Positional references: `GROUP BY 1` (means the first SELECT column)
- Full expressions: `GROUP BY EXTRACT(year FROM created_at)`

**5. Bind aggregate expressions in SELECT**  
`SUM(price)`, `COUNT(DISTINCT customer_id)`, `AVG(age)` — these are bound
after GROUP BY because the binder needs to know the grouping to validate that
non-aggregated columns in SELECT are also in GROUP BY.

**6. Bind HAVING**  
Similar to WHERE but runs after aggregation. Can reference aggregate functions.

```sql
HAVING SUM(price) > 10000
```
→ `SUM(price)` → aggregate function, type DECIMAL.

**7. Bind QUALIFY**  
A DuckDB extension: filter rows using window function results.

```sql
QUALIFY RANK() OVER (PARTITION BY region ORDER BY sales DESC) = 1
```

**8. Bind ORDER BY**  
Can reference SELECT aliases, column names, or positional numbers.

**9. Bind LIMIT and OFFSET**  
These must be constant expressions (or parameters).

**10. Expand SELECT ***  
`SELECT *` is expanded to an explicit list of all columns from all tables in
scope. `SELECT orders.*` expands only the orders table's columns.
`SELECT * EXCLUDE (password)` expands then removes the listed columns.

**Result:** A `BoundSelectNode` — a fully typed version of the SELECT with
every column reference resolved to a `(table_index, column_index)` pair.

### `bind_setop_node.cpp` (~12 KB) — UNION / INTERSECT / EXCEPT

Handles set operations. Key rules:
- Both sides must produce the **same number of columns**.
- Corresponding columns must have **compatible types** (the binder finds a
  common type and inserts casts if needed — e.g., INTEGER and BIGINT → BIGINT).
- Column names come from the **left side** only.

```sql
SELECT name, age FROM employees
UNION ALL
SELECT name, age FROM contractors
```
→ Both produce 2 columns: (name:VARCHAR, age:INTEGER). Compatible. OK.

### `bind_cte_node.cpp` — Common Table Expressions (WITH ... AS)

A CTE is a named subquery. Binding:
1. Registers the CTE name in the current scope.
2. Binds the CTE's body query (which becomes a full bound query plan).
3. When the CTE name appears in FROM clauses, it is replaced with a reference
   to the already-bound CTE.

For **non-materialized CTEs**: the CTE body may be inlined (copy-pasted) into
each place it is used, allowing further optimization.

For **materialized CTEs** (`WITH ... AS MATERIALIZED`): the CTE is computed
once and stored. References become `LogicalCTERef` operators pointing to the
stored result.

### `bind_recursive_cte_node.cpp` (~13 KB) — RECURSIVE CTEs

```sql
WITH RECURSIVE hierarchy AS (
    SELECT id, name, NULL AS parent FROM employees WHERE manager_id IS NULL  -- base case
    UNION ALL
    SELECT e.id, e.name, h.name FROM employees e
    JOIN hierarchy h ON e.manager_id = h.id                                  -- recursive case
)
SELECT * FROM hierarchy
```

Recursive CTEs are significantly more complex:
1. Bind the **base case** (the non-recursive SELECT).
2. Register the CTE name with the type information from the base case.
3. Bind the **recursive case** (which references the CTE itself).
4. Validate that both sides have compatible column types.
5. Create a `LogicalRecursiveCTE` operator with both sides.

### `plan_select_node.cpp` and `plan_query_node.cpp` — Building the Plan

After binding, these files convert the `BoundSelectNode` into a tree of
`LogicalOperator` objects:

1. Start with a `LogicalGet` (table scan) for each FROM table.
2. If there are JOINs, connect them into a `LogicalJoin` tree.
3. If there is a WHERE clause, add a `LogicalFilter` on top.
4. If there is a GROUP BY, add a `LogicalAggregate`.
5. If there is a HAVING, add another `LogicalFilter`.
6. Add a `LogicalProjection` for the SELECT list.
7. If there is an ORDER BY, add a `LogicalOrder`.
8. If there is LIMIT/OFFSET, add a `LogicalLimit` (or `LogicalTopN` if both
   ORDER BY and LIMIT are present — can be handled more efficiently together).
9. If there is DISTINCT, add a `LogicalDistinct`.

---

## 6. Table Reference Binding (`binder/tableref/`) — The FROM Clause

### `bind_basetableref.cpp` (~13 KB) — Plain Table Names

`FROM orders` or `FROM main.orders`.

1. Look up the table name in the catalog.
2. If not found: check if it is a CTE name. If yes, reference the CTE.
3. If still not found: check if it is a view. If yes, expand the view (bind the
   view's definition as a subquery).
4. If found as a real table: create a `LogicalGet` (table scan) operator.
5. Register the table and its columns in the bind context.

**View expansion:** When a view is referenced, the binder:
- Creates a **child Binder** to bind the view's body in the view's original
  context (not the calling query's context).
- Uses `bound_views` to detect circular view references (e.g., view A references
  view B which references view A).

### `bind_joinref.cpp` (~14 KB) — JOIN

```sql
orders JOIN customers ON orders.customer_id = customers.id
```

1. Bind the left side (recursively — it might be another join).
2. Bind the right side.
3. Both sides' columns are now registered in the bind context.
4. Bind the ON condition — `orders.customer_id = customers.id` → resolves both
   columns.
5. For `JOIN ... USING (column)` syntax: create equality conditions for each
   named column.
6. For `NATURAL JOIN`: automatically create equality conditions for all columns
   with the same name in both tables.
7. Create a `LogicalJoin` with the bound condition.

### `plan_joinref.cpp` (~18 KB) — Choosing the Join Algorithm

After binding, this file decides the physical nature of the join:

- **INNER JOIN with equality condition** → becomes a hash join candidate for
  the optimizer.
- **CROSS JOIN** (no condition) → becomes a `LogicalCrossProduct`.
- **LEFT/RIGHT/FULL OUTER JOIN** → becomes a `LogicalComparisonJoin` with the
  join type noted.
- **Multiple tables with no explicit join** → `FROM a, b` is a cross product.

### `bind_subqueryref.cpp` — Subquery in FROM

```sql
FROM (SELECT region, SUM(sales) AS total FROM orders GROUP BY region) AS totals
```

1. Creates a **child Binder** for the subquery's scope.
2. Binds the inner query fully (recursive call to the whole binding pipeline).
3. Registers the subquery's output columns (with the given alias `totals`) in
   the outer bind context.

### `bind_table_function.cpp` (~20 KB) — Table-Valued Functions

```sql
FROM read_csv('data.csv', header=true)
FROM range(1, 100)
FROM generate_series(0, 10, 2)
```

1. Looks up the function name in the function catalog.
2. Binds the arguments (evaluates constant arguments at bind time).
3. Calls the function's **bind function** — a special method that, given the
   arguments, returns the output schema (column names and types). For example,
   `read_csv` opens the file and peeks at the header to determine column names.
4. Creates a `LogicalGet` with the table function and its bound arguments.
5. Registers the output columns in the bind context.

### `bind_pivot.cpp` (~41 KB) — PIVOT

PIVOT is the most complex table reference. It rotates rows into columns:

```sql
FROM sales PIVOT (SUM(amount) FOR region IN ('East', 'West', 'North'))
```

Becomes equivalent to:
```sql
SELECT
    SUM(CASE WHEN region = 'East' THEN amount END) AS East,
    SUM(CASE WHEN region = 'West' THEN amount END) AS West,
    SUM(CASE WHEN region = 'North' THEN amount END) AS North
FROM sales
```

The binder rewrites the PIVOT into this equivalent query form before binding.
If the pivot values are not known at bind time (dynamic pivot), it must first
execute a query to discover them.

### `bind_expressionlistref.cpp` — VALUES Clause

```sql
FROM (VALUES (1, 'Alice'), (2, 'Bob')) AS t(id, name)
```

1. Binds each expression in each row.
2. Finds the common type for each column (if row 1 has an integer and row 2 has
   a float for the same column, promotes to float).
3. Inserts type casts where needed.
4. Registers the output columns.

---

## 7. Expression Binding (`binder/expression/`) — One File Per Expression Type

Each expression type has its own binding file. Binding an expression means:
- Resolving any column references it contains.
- Calling the correct function/operator.
- Determining the output type.
- Inserting implicit type casts where needed.

### `bind_columnref_expression.cpp` — Column References

`price`, `orders.price`, `main.orders.price`.

1. Call `bind_context.GetMatchingBinding(column_name)`.
2. If found: create `BoundColumnRefExpression` with the table index, column
   index, and the column's type.
3. If not found in current scope: walk up parent binders (correlated column).
   Each level walked increments the `depth` field.
4. If not found anywhere: error "column not found."

**Ambiguity resolution:** If both `orders` and `order_items` have a column
named `id`, `bind_context` reports an ambiguity error and suggests qualifying
the name.

### `bind_function_expression.cpp` (~23 KB) — Function Calls

`SUM(price)`, `UPPER(name)`, `date_diff('day', start_date, end_date)`.

This is the most complex expression binder because functions come in many forms:

**Step 1 — Bind the arguments.** Each argument expression is bound recursively.

**Step 2 — Look up the function.**  
The function catalog is searched by name. Functions are overloaded (the same
name can have different behaviour for different argument types), so the lookup
uses the **argument types** to find the best match.

**Step 3 — Type coercion.**  
If no exact match exists, find the closest match and insert implicit CASTs.
For example, `SUM(integer_col)` where the table has `price` as SMALLINT:
the catalog may find `SUM(BIGINT)` as the best match and insert
`CAST(price AS BIGINT)`.

**Step 4 — Classify the function:**
- **Scalar function** (one output per row, like `UPPER`) → `BoundFunctionExpression`.
- **Aggregate function** (one output per group, like `SUM`) → `BoundAggregateExpression`.
- **Window function** (with OVER clause, like `RANK()`) → `BoundWindowExpression`.
- **Macro** (user-defined with `CREATE MACRO`) → inline the macro body.

**Step 5 — Call the function's own bind function.**  
Many functions have a custom **bind function** that runs at plan time to
determine the output type or validate arguments. For example:
- `printf('%s has %d items', name, count)` → bind function checks the format
  string and returns VARCHAR.
- `struct_extract(struct_col, 'field_name')` → bind function looks up the
  field in the struct type and returns its type.

### `bind_aggregate_expression.cpp` (~13 KB) — Aggregate Functions

`SUM(price)`, `COUNT(DISTINCT customer_id)`, `AVG(age)`.

Extra binding steps for aggregates:
1. Verify the aggregate is **not nested** inside another aggregate (invalid SQL).
2. Bind the `FILTER` clause if present: `COUNT(*) FILTER (WHERE status = 'paid')`.
3. Bind the `ORDER BY` inside the aggregate if present:
   `STRING_AGG(name ORDER BY age DESC)`.
4. For `DISTINCT` aggregates: validate the argument list.
5. Register the aggregate with the select binder so it can check GROUP BY
   validity.

### `bind_window_expression.cpp` (~16 KB) — Window Functions

`RANK() OVER (PARTITION BY region ORDER BY sales DESC)`.

1. Bind the function arguments (same as regular function).
2. Bind the PARTITION BY expressions.
3. Bind the ORDER BY expressions.
4. Bind the frame specification (`ROWS BETWEEN 3 PRECEDING AND CURRENT ROW`).
5. Validate the combination (e.g., not all window functions support all frame
   types).
6. Create `BoundWindowExpression`.

### `bind_star_expression.cpp` (~15 KB) — SELECT *

`*`, `orders.*`, `* EXCLUDE (password)`, `* REPLACE (price * 1.1 AS price)`.

This is not really an expression after binding — `*` is expanded into an
explicit list of column references:

1. Find all columns currently visible in the bind context.
2. Apply any EXCLUDE list (remove named columns).
3. Apply any REPLACE list (substitute specific columns with new expressions).
4. Return a `BoundExpandedExpression` that the SELECT list can flatten.

### `bind_subquery_expression.cpp` — Subqueries in Expressions

```sql
WHERE salary > (SELECT AVG(salary) FROM employees)
WHERE customer_id IN (SELECT id FROM premium_customers)
WHERE EXISTS (SELECT 1 FROM alerts WHERE alerts.user_id = users.id)
```

1. Create a child Binder for the subquery scope.
2. Bind the inner SELECT fully.
3. Classify the subquery type:
   - **Scalar subquery**: must return exactly one row and one column.
   - **EXISTS / NOT EXISTS**: just checks whether any rows exist.
   - **IN / NOT IN**: checks membership.
   - **ANY / ALL**: compares with comparison operator.
4. Mark any outer-scope column references as correlated (depth > 0).
5. The resulting subquery will later be decorrelated by `flatten_dependent_join.cpp`.

### `bind_case_expression.cpp` — CASE WHEN

```sql
CASE WHEN age < 18 THEN 'minor' WHEN age >= 65 THEN 'senior' ELSE 'adult' END
```

1. Bind each WHEN condition (must produce BOOLEAN type).
2. Bind each THEN result.
3. Bind the ELSE result (or NULL if absent).
4. Find the **common type** across all THEN/ELSE results (promote if needed).
5. Insert type casts to bring all branches to the common type.

### `bind_cast_expression.cpp` — Type Casts

`CAST(price AS INTEGER)`, `price::INTEGER`.

1. Bind the inner expression.
2. Look up whether a cast from the source type to the target type exists.
3. If yes: create `BoundCastExpression`.
4. If no valid cast exists: error "cannot cast TYPE_A to TYPE_B."

**TRY_CAST:** Binds the same way but marks the cast as "try" — on failure,
return NULL instead of an error.

### `bind_operator_expression.cpp` (~11 KB) — Unary/Binary Operators

`NOT condition`, `value IS NULL`, `value IS NOT NULL`, `value IN (list)`.

These are internally treated as special function calls:
- `NOT x` → `OP_NOT(x)` → looks up the `!` operator in the catalog.
- `x IS NULL` → `ISNULL(x)` → special operator returning BOOLEAN.
- `x IN (1, 2, 3)` → expanded into `x = 1 OR x = 2 OR x = 3` (for small lists)
  or a special `IN` operator for larger lists.

### `bind_comparison_expression.cpp` (~7 KB) — Comparisons

`=`, `<>`, `<`, `>`, `<=`, `>=`.

1. Bind both sides.
2. Find the common type for comparison (e.g., comparing INTEGER and BIGINT →
   both promoted to BIGINT).
3. Insert type casts if needed.
4. Create `BoundComparisonExpression`.

### `bind_lambda.cpp` (~11 KB) — Lambda Expressions

`list_transform([1, 2, 3], x -> x * 2)`.

1. Determine the lambda parameter type(s) from the list element type.
2. Register the lambda parameter(s) as special column references in scope.
3. Bind the lambda body expression (`x * 2`).
4. Create `BoundLambdaExpression`.

---

## 8. Expression Binder Specializations (`expression_binder/`) — Clause Context

Different SQL clauses have different rules about what expressions they allow.
For example, window functions are not allowed in WHERE, and aggregates are not
allowed in GROUP BY (in most databases). DuckDB enforces these rules with
**clause-specific expression binders** — they wrap `ExpressionBinder` but
override certain checks.

| File | Used for | Special rules |
|------|----------|---------------|
| `where_binder.cpp` | WHERE clause | No aggregates, no window functions |
| `having_binder.cpp` | HAVING clause | Aggregates OK; non-aggregate columns must be in GROUP BY |
| `group_binder.cpp` | GROUP BY clause | Resolves aliases from SELECT; no aggregates |
| `select_binder.cpp` | SELECT list | Aggregates and window functions OK |
| `order_binder.cpp` | ORDER BY clause | Resolves aliases from SELECT; positional refs OK |
| `qualify_binder.cpp` | QUALIFY clause | Window functions required |
| `projection_binder.cpp` | SELECT projection | Final expression binding for output |
| `update_binder.cpp` | UPDATE SET | No aggregates |
| `insert_binder.cpp` | INSERT VALUES | Type checking against column types |
| `check_binder.cpp` | CHECK constraints | Must produce BOOLEAN |
| `constant_binder.cpp` | Default values, LIMIT | Only constants and parameters |
| `index_binder.cpp` | CREATE INDEX | Only deterministic expressions |
| `lateral_binder.cpp` | LATERAL subqueries | Outer scope columns visible |
| `returning_binder.cpp` | RETURNING clause | Access to inserted/updated columns |
| `table_function_binder.cpp` | Table function args | Named parameter resolution |

---

## 9. Bound Expressions (`expression/`) — The Output of Binding

After binding, every parsed expression becomes a **bound expression** — an
object that knows its exact output type and refers to tables/columns by index
rather than by name.

### The most important: `bound_columnref_expression.cpp`

```
BoundColumnRefExpression {
    return_type = DECIMAL(10, 2)      ← The column's data type
    binding = ColumnBinding(
        table_index = 0,              ← "This is from the first table in the plan"
        column_index = 3              ← "Specifically the 4th column"
    )
    depth = 0                         ← 0 means same scope; >0 means correlated
}
```

The `ColumnBinding(table_index, column_index)` pair is the universal way DuckDB
refers to columns throughout the entire planning and optimization pipeline. It
replaces the original string name completely.

### Other bound expression types

| Class | Contains |
|-------|---------|
| `BoundConstantExpression` | A literal value (`42`, `'hello'`, `NULL`) with its type |
| `BoundFunctionExpression` | Function name, bound argument list, output type, function pointer |
| `BoundAggregateExpression` | Aggregate function, bound arguments, DISTINCT flag, FILTER, ORDER BY |
| `BoundWindowExpression` | Window function, partition by, order by, frame bounds |
| `BoundCastExpression` | The inner expression, target type, whether it is TRY_CAST |
| `BoundCaseExpression` | List of (condition, result) pairs, ELSE expression, common type |
| `BoundConjunctionExpression` | List of children connected by AND or OR |
| `BoundComparisonExpression` | Left, right, operator type |
| `BoundOperatorExpression` | Operator type (IS NULL, NOT, etc.), child expressions |
| `BoundSubqueryExpression` | The bound inner query plan, subquery type |
| `BoundParameterExpression` | Parameter index, type (set at execute time) |
| `BoundLambdaExpression` | Lambda parameter bindings, body expression |
| `BoundUnnestExpression` | The list expression being unnested |

---

## 10. Logical Operators (`operator/`) — The Plan Nodes

A **logical operator** is one step in the query plan. Operators form a tree —
each operator reads from its children and produces output rows. The leaves of
the tree are table scans; the root is whatever is computed last (usually a
projection).

### `logical_get.cpp` (~16 KB) — Table Scan

The leaf of almost every plan tree. Represents reading data from a source.

```
LogicalGet {
    table_index = 0              ← Unique ID for this table in the plan
    function = table_scan_fn    ← The function that performs the actual scan
    bind_data = ...             ← Configuration (which table, which file, etc.)
    column_ids = [0, 3, 5]     ← Only scan these columns (column pruning)
    table_filters = ...         ← Filters pushed down to the scan level
    projected_input = ...       ← Column projections
}
```

Every table source uses `LogicalGet`, regardless of whether it is:
- A real DuckDB table.
- A CSV file (`read_csv`).
- A Parquet file (`read_parquet`).
- A table function (`range`, `generate_series`).
- A Postgres table via the postgres extension.

The difference is in the `function` and `bind_data` fields.

### `logical_filter.cpp` — WHERE and HAVING

```
LogicalFilter {
    expressions = [ age > 30, status = 'active' ]
    children = [ LogicalGet(orders) ]
}
```

Takes its child's output and keeps only rows where all expressions are true.
Multiple conditions are all applied together.

### `logical_projection.cpp` — SELECT List

```
LogicalProjection {
    expressions = [ name, price * 1.1, UPPER(city) ]
    children = [ LogicalFilter(...) ]
}
```

Takes its child's rows and computes new columns. Discards columns not needed
downstream.

### `logical_aggregate.cpp` — GROUP BY and Aggregates

```
LogicalAggregate {
    groups = [ region, year ]        ← GROUP BY columns
    expressions = [ SUM(price), COUNT(*) ]  ← Aggregate functions
    children = [ LogicalFilter(...) ]
}
```

Groups rows and computes one aggregate result per group.

### `logical_join.cpp` and variants — JOIN

```
LogicalComparisonJoin {
    join_type = INNER
    conditions = [ orders.customer_id = customers.id ]
    children = [
        LogicalGet(orders),
        LogicalGet(customers)
    ]
}
```

Join variants:
- `LogicalComparisonJoin` — for joins with equality/comparison conditions.
- `LogicalCrossProduct` — for CROSS JOIN (no condition).
- `LogicalAnyJoin` — for joins with arbitrary conditions.
- `LogicalPositionalJoin` — for POSITIONAL JOIN (match by row number).

### `logical_order.cpp` — ORDER BY

```
LogicalOrder {
    orders = [ (total, DESCENDING), (name, ASCENDING) ]
    children = [ LogicalProjection(...) ]
}
```

### `logical_limit.cpp` — LIMIT / OFFSET

```
LogicalLimit {
    limit = 10
    offset = 20
    children = [ LogicalOrder(...) ]
}
```

### `logical_top_n.cpp` — ORDER BY + LIMIT Together

When a query has both ORDER BY and LIMIT, they can often be handled together
more efficiently than sorting everything and then taking the top N:

```
LogicalTopN {
    orders = [ (total, DESCENDING) ]
    limit = 10
    children = [ LogicalProjection(...) ]
}
```

### `logical_window.cpp` — Window Functions

```
LogicalWindow {
    window_expressions = [ RANK() OVER (PARTITION BY region ORDER BY sales) ]
    children = [ LogicalProjection(...) ]
}
```

### `logical_distinct.cpp` — DISTINCT

Eliminates duplicate rows from the output.

### `logical_set_operation.cpp` — UNION / INTERSECT / EXCEPT

```
LogicalSetOperation {
    type = UNION_ALL
    children = [
        LogicalGet(employees),
        LogicalGet(contractors)
    ]
}
```

### `logical_insert.cpp`, `logical_update.cpp`, `logical_delete.cpp`

These wrap a SELECT plan (the source data) and add write operations to the leaf:

```
LogicalInsert {
    table = "orders"
    children = [ LogicalProjection(...) ]   ← The data to insert
}
```

### `logical_create_table.cpp`, `logical_create_index.cpp`

DDL operations. `LogicalCreateTable` holds the full table definition.
`LogicalCreateIndex` holds the index expression and which table/column to index.

### `logical_explain.cpp` — EXPLAIN

Wraps any other logical operator and signals that instead of executing the plan,
the executor should print a description of it.

### `logical_sample.cpp` — SAMPLE Clause

```sql
SELECT * FROM orders TABLESAMPLE (1%)
```

Wraps the scan and randomly keeps approximately 1% of rows.

### `logical_recursive_cte.cpp` and `logical_materialized_cte.cpp`

Special operators for common table expressions that are referenced more than
once or that are recursive.

### `logical_copy_to_file.cpp` — COPY TO

Wraps a SELECT plan and writes results to a file instead of returning them to
the client.

### `logical_pivot.cpp` — PIVOT

After the PIVOT rewrite (see `bind_pivot.cpp`), this operator represents the
pivoted scan.

---

## 11. `subquery/flatten_dependent_join.cpp` — Subquery Decorrelation

**File size:** ~46 KB — the largest single file in the planner.  
**What it does:** This is one of the most important transformations in the
entire system. It converts **correlated subqueries** into **joins**, which are
dramatically faster.

### What is a correlated subquery?

```sql
SELECT name, salary
FROM employees e
WHERE salary > (SELECT AVG(salary) FROM employees WHERE department = e.department)
```

The inner query `SELECT AVG(salary) FROM employees WHERE department = e.department`
references `e.department` from the outer query. This is a **correlated
subquery** — the inner query must be re-executed for every row of the outer
query.

If there are 10,000 employees in 50 departments, the naive execution runs the
inner query 10,000 times — once per employee row. Very slow.

### The decorrelation transformation

`FlattenDependentJoins` rewrites this into an equivalent query that can be
executed as a **single join**:

```sql
SELECT e.name, e.salary
FROM employees e
JOIN (SELECT department, AVG(salary) AS avg_sal FROM employees GROUP BY department) dept_avg
ON e.department = dept_avg.department
WHERE e.salary > dept_avg.avg_sal
```

Now the average-per-department is computed once and joined in. The plan runs in
O(n) instead of O(n²).

### How it works

The transformation walks the logical plan looking for `LogicalDependentJoin`
nodes (which represent correlated subqueries in the plan). For each one:

1. **Collect correlated columns** — find all outer-scope column references in
   the subquery (the `depth > 0` ones).

2. **Push the correlation up** — rewrite the subquery's plan to explicitly pass
   the correlated columns as extra GROUP BY columns, turning the correlation
   into regular data flow.

3. **Replace with a join** — replace the `LogicalDependentJoin` with a
   `LogicalComparisonJoin` that joins on the correlated column values.

4. **Propagate projections** — ensure all columns needed downstream are still
   available after the rewrite.

This transformation is mathematically proven to produce equivalent results.
It works for EXISTS, IN, scalar subqueries, and ANY/ALL expressions.

---

## 12. `filter/` — Table Filters

The filter subsystem lets DuckDB push filter conditions all the way down to the
table scan level — into the storage engine — so that rows are discarded before
they even enter the query engine.

### `expression_filter.cpp` (~22 KB)

Converts a bound filter expression into a `TableFilter` object that the storage
engine can evaluate directly on a column segment without decompressing all data.

Examples:
- `age > 30` → `ConstantFilter(GREATER_THAN, 30)` pushed to the scan.
- `name IS NOT NULL` → `NullFilter(IS_NOT_NULL)` pushed to the scan.
- `status IN ('A', 'B')` → `InFilter(['A', 'B'])` pushed to the scan.
- `region = 'East' AND year = 2024` → `ConjunctionFilter` combining both.

### Filter types

| Filter | Used for |
|--------|---------|
| `ConstantFilter` | Comparison to a constant: `col > 30` |
| `NullFilter` | IS NULL or IS NOT NULL |
| `InFilter` | `col IN (value_list)` |
| `ConjunctionFilter` | AND of multiple filters |
| `BloomFilter` | Semi-join filter: "only keep rows whose key appears in the hash table from the other side" |
| `DynamicFilter` | Filters built at runtime (e.g., from join probe phase) |
| `PrefixRangeFilter` | `col LIKE 'prefix%'` → range filter on the stored bytes |
| `StructFilter` | Filter on a field inside a struct column |

These filters are passed to `LogicalGet` and ultimately to the storage engine's
`ColumnData::ScanFilter()` method, which evaluates them during segment
decompression.

---

## 13. `column_qualifier.cpp` and `statement_preprocessor.cpp`

### `column_qualifier.cpp` (~21 KB)

Before binding, some column references need to be **qualified** — given their
full table name. For example, in:
```sql
SELECT name FROM orders WHERE name = 'widget'
```
both `name` references must be qualified to `orders.name` to avoid ambiguity.

This file walks the AST and adds table qualifiers where they are missing, using
the visible tables in the current scope.

### `statement_preprocessor.cpp` (~8 KB)

Runs before binding to handle some statement-level transformations:
- **Macro expansion** — replaces `CREATE MACRO` calls with their definitions.
- **Star expansion hints** — expands some `SELECT *` patterns early.
- **Named parameter normalization** — converts `$name` to positional parameters
  where needed.

---

## 14. End-to-End Example: Planning a SELECT Query

Let's trace exactly what happens when DuckDB plans:

```sql
SELECT region, SUM(price) AS total
FROM orders
WHERE status = 'paid'
GROUP BY region
HAVING SUM(price) > 1000
ORDER BY total DESC
LIMIT 5
```

### Step 1 — `planner.cpp` calls `binder.Bind(select_statement)`

### Step 2 — `bind_select.cpp` calls `binder.Bind(*select_stmt.node)`

### Step 3 — `bind_select_node.cpp` binds all clauses

**FROM clause:**
- `orders` → found in catalog → `LogicalGet(orders, table_index=0)` planned.
- Columns registered in bind context:
  - `orders.id` → (0, 0), type INTEGER
  - `orders.status` → (0, 1), type VARCHAR
  - `orders.price` → (0, 2), type DECIMAL
  - `orders.region` → (0, 3), type VARCHAR
  - ...

**WHERE clause (`status = 'paid'`):**
- `status` → resolved to (0, 1), VARCHAR.
- `'paid'` → ConstantExpression, VARCHAR.
- `=` → `BoundComparisonExpression(EQUAL, col(0,1), 'paid')`.
- Pushed down to a `TableFilter` → the storage engine will only return rows
  where `status = 'paid'`.

**GROUP BY (`region`):**
- `region` → resolved to (0, 3), VARCHAR.
- Registered as grouping column.

**SELECT list (`region`, `SUM(price) AS total`):**
- `region` → `BoundColumnRefExpression(0, 3, VARCHAR)`.
- `SUM(price)` → `price` resolved to (0, 2) → `BoundAggregateExpression(SUM, [col(0,2)], DECIMAL)`.
- Alias `total` registered.

**HAVING (`SUM(price) > 1000`):**
- `SUM(price)` → same aggregate as above (reused).
- `1000` → constant, cast to DECIMAL.
- `>` → `BoundComparisonExpression(GREATER_THAN, SUM(col(0,2)), 1000)`.

**ORDER BY (`total DESC`):**
- `total` → resolved as alias for `SUM(price)` → `BoundColumnRefExpression` pointing to aggregate output.
- Direction: DESCENDING.

**LIMIT (5):**
- Constant 5.

### Step 4 — `plan_select_node.cpp` builds the logical plan

```
LogicalTopN (ORDER BY total DESC, LIMIT 5)
  └── LogicalFilter (HAVING: SUM(price) > 1000)
        └── LogicalAggregate (GROUP BY region; SUM(price) AS total)
              └── LogicalProjection (region, price)
                    └── LogicalFilter (WHERE: status = 'paid'  ← pushed to scan)
                          └── LogicalGet (orders, col_ids=[1,2,3])
```

Note:
- The WHERE filter was pushed all the way to `LogicalGet` as a `TableFilter`.
  The storage engine will skip rows where `status ≠ 'paid'` without even
  entering the query engine.
- Only columns 1 (status), 2 (price), and 3 (region) are scanned — column 0
  (id) is never read.
- ORDER BY and LIMIT are combined into a single `LogicalTopN` — more efficient
  than sorting everything and then taking 5.

### Step 5 — `flatten_dependent_join.cpp`

No correlated subqueries in this query. Nothing to do.

### Step 6 — `ColumnBindingResolver::Verify()`

Walks the entire plan tree and verifies every `ColumnBinding` is valid.
All `(table_index, column_index)` pairs confirmed correct.

### Step 7 — Plan returned to the optimizer

The plan is correct and verified. The optimizer will now try to make it faster
(reordering joins, choosing physical operators, etc.).

---

## Summary Table

| File / Folder | Role |
|--------------|------|
| `planner.cpp` | Top-level coordinator: calls binder, decorrelates, verifies |
| `binder.cpp` | Main name resolver: dispatches to statement-specific binders |
| `bind_context.cpp` | Scope table: tracks visible tables and columns |
| `binder/statement/*.cpp` | One binder per SQL statement type |
| `binder/query_node/bind_select_node.cpp` | Binds all SELECT clauses in correct order |
| `binder/query_node/plan_*.cpp` | Converts bound nodes into logical operator trees |
| `binder/tableref/*.cpp` | Binds FROM clause (tables, joins, subqueries, functions) |
| `binder/expression/*.cpp` | Binds each expression type (columns, functions, comparisons, etc.) |
| `expression_binder/*.cpp` | Clause-specific binding rules (WHERE ≠ HAVING ≠ SELECT) |
| `expression/*.cpp` | Bound expression objects (with types and column binding indices) |
| `operator/*.cpp` | Logical operator nodes forming the plan tree |
| `subquery/flatten_dependent_join.cpp` | Converts correlated subqueries into joins |
| `filter/*.cpp` | Table-level filters pushed to the storage scan |
| `column_qualifier.cpp` | Adds table qualifiers to ambiguous column names |
| `statement_preprocessor.cpp` | Pre-binding transformations (macros, star hints) |
