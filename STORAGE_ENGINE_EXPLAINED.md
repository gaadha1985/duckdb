# DuckDB Storage Engine — Explained Line by Line

This document walks through every major component of DuckDB's storage engine
(`src/storage/`) in plain language. No programming background is assumed.

---

## What Is a Storage Engine?

Before reading any code, it helps to know what a storage engine does. Think of
it as the warehouse manager of a database:

- It decides **where** data lives on disk.
- It decides **how** data is packed to save space.
- It makes sure data is **not lost** if the computer crashes.
- It decides **what to keep in memory** so reads are fast.
- It makes sure two people changing data at the same time **don't corrupt each other's work**.

Every file in `src/storage/` is responsible for one piece of that job.

---

## The Big Map: How All the Pieces Fit Together

```
Your data
    |
    v
[ StorageManager ]         ← The supervisor. Knows about the database file.
    |
    |---> [ WriteAheadLog ]     ← Safety log. Writes changes here first.
    |
    |---> [ BlockManager ]      ← Manages fixed-size chunks of the file.
    |         |
    |         └---> [ BufferManager ]   ← Decides what stays in memory.
    |                   |
    |                   └---> [ BufferPool ]  ← The memory pool itself.
    |
    |---> [ CheckpointManager ]  ← Periodically saves a clean snapshot.
    |
    └---> [ DataTable ]          ← The actual table data.
              |
              └---> [ RowGroupCollection ]
                        |
                        └---> [ RowGroup ]
                                  |
                                  └---> [ ColumnData ]
                                            |
                                            └---> [ ColumnSegment + Compression ]
```

Each arrow means "is made of" or "depends on". Let's go through each box from
the top down.

---

## 1. `storage_manager.cpp` — The Supervisor

**File size:** ~29 KB  
**What it does:** This is the first thing that runs when DuckDB opens or creates
a database. It is the top-level manager that sets everything else up.

### Opening a new database (line-by-line story)

When you say "open database file `my_data.db`", this file runs:

1. **Check if the file exists.**
   - If it does not exist → create a brand-new empty database.
   - If it does exist → load the existing data.

2. **Read the "magic bytes"** — the very first few bytes of the `.db` file.
   These are a special signature (like a file fingerprint) that prove "yes, this
   is a real DuckDB file and not a random file someone accidentally opened."
   (`magic_bytes.cpp` handles this check.)

3. **Check the storage version number.** DuckDB's file format has evolved over
   time. The version number stored in the file tells DuckDB whether it can read
   this file or whether it is too old/new.

4. **Set up encryption** (if the user configured a password). DuckDB can
   encrypt the entire database file using AES-GCM or AES-CTR ciphers.

5. **Create the BlockManager.** The block manager is handed the open file so
   it can start reading and writing fixed-size chunks.

6. **Replay the WAL (Write-Ahead Log)** — if the computer crashed last time,
   any uncommitted changes are replayed here. (More on WAL below.)

7. **Hand control to the rest of the system.**

### Key methods

- `Initialize(QueryContext)` — Entry point. Calls everything else.
- `LoadDatabase(QueryContext)` — Decides create-new vs. open-existing.
- `CreateCheckpoint()` — Triggers a full save of all data to disk.
- `GetWAL()` — Returns the Write-Ahead Log object for writing transactions.

### The two modes

DuckDB can run in two modes:

- **Persistent mode** (`SingleFileStorageManager`) — data is saved to a `.db`
  file. This is what you use when you want data to survive after the program
  closes.
- **In-memory mode** — all data lives in RAM. Faster, but lost when the program
  exits.

---

## 2. `write_ahead_log.cpp` and `wal_replay.cpp` — The Safety Log

**What it does:** Imagine you are writing a very important letter and the power
goes out halfway through. Without a safety system, your letter is ruined. The
Write-Ahead Log (WAL) is DuckDB's safety system.

### The rule: write to the log first

Every time data is changed (a row inserted, updated, or deleted), DuckDB writes
a record of that change to the WAL *before* changing the actual data on disk.
The WAL is an append-only file — nothing is ever overwritten, only added.

This means:
- If the computer crashes *before* the main data is updated, the WAL has the
  record and DuckDB can finish the job when it restarts.
- If the computer crashes *after* the main data is updated, DuckDB knows the
  transaction completed and the WAL record can be ignored.

### `write_ahead_log.cpp` — Writing to the log

Every transaction writes entries like:
- `WAL_INSERT_TUPLE` — a new row was inserted, here is the data.
- `WAL_UPDATE` — a row was changed, here is the old and new value.
- `WAL_DELETE` — a row was deleted.
- `WAL_CREATE_TABLE` — a new table was created.
- `WAL_CHECKPOINT` — a clean snapshot was written; entries before this point
  can be discarded.

Each entry is written sequentially, one after another, like pages in a diary.

### `wal_replay.cpp` — Replaying the log on startup

When DuckDB opens a file and finds an unfinished WAL, `wal_replay.cpp` reads
every entry from the beginning and re-applies them in order. This brings the
database back to the state it was in just before the crash.

- **Committed transactions** (which wrote a `COMMIT` entry) are applied.
- **Uncommitted transactions** (which never wrote `COMMIT`) are ignored —
  as if they never happened.

---

## 3. `block_manager.cpp` / `single_file_block_manager.cpp` — The Filing System

**What it does:** The database file on disk is divided into equal-size
**blocks** (typically 256 KB each). The block manager is the librarian that
assigns a unique ID to each block, knows which blocks are in use, and reads/
writes individual blocks.

Think of blocks like pages in a notebook. Every piece of data must live on one
or more pages. The block manager assigns page numbers and keeps a directory
of which pages contain what.

### `block_manager.hpp` — The abstract contract

This is an interface (a blueprint) that says: "any block manager must be able
to do these things":
- `ReadBlock(block_id, destination)` — read block #X from disk into memory.
- `WriteBlock(block_id, source)` — write memory to block #X on disk.
- `AllocateBlock()` — reserve a new block and return its ID.
- `MarkBlockAsModified(block_id)` — this block has unsaved changes.
- `MarkBlockAsCheckpointed(block_id)` — this block is safely on disk.

### `single_file_block_manager.cpp` (~54 KB) — The real implementation

This is where the actual file I/O happens. It manages a single `.db` file and:

1. Keeps a **free list** — a list of block IDs that have been deleted and can
   be reused, like recycling paper.
2. Tracks which blocks are **in use** vs. **free** with a bitmask.
3. Writes the free list and bitmask to disk as part of every checkpoint so the
   information survives crashes.
4. Supports **two checkpoint slots** (called `checkpoint_A` and `checkpoint_B`)
   — DuckDB alternates between them so there is always one valid checkpoint
   even if writing the new one is interrupted.

---

## 4. `buffer_manager.cpp` and `standard_buffer_manager.cpp` — The Memory Manager

**What it does:** Disk is slow; RAM is fast. The buffer manager is the traffic
controller that decides which blocks from disk are loaded into memory, and
which ones get evicted (removed from memory) when memory runs out.

### The core idea: pins and unpins

Every time part of the code wants to read or write a block, it **pins** it:

```
pin(block_42) → "I need block 42 in memory right now, do not evict it."
```

When it is done:
```
unpin(block_42) → "I am done with block 42, you may evict it if needed."
```

A pinned block can never be removed from memory. An unpinned block is a
candidate for eviction when memory is tight.

### `buffer/buffer_pool.cpp` (~22 KB) — The eviction engine

When memory is full and a new block needs to be loaded, something must be
removed. The buffer pool uses a priority queue of unpinned blocks sorted by
"how recently they were used." The least-recently-used block gets evicted first.

Eviction tiers (priority order — DuckDB tries to evict cheapest first):
1. **BLOCK / EXTERNAL_FILE blocks** — large blocks loaded from disk.
   Evicting them just means releasing the memory; the data is already on disk.
2. **MANAGED_BUFFER** — medium-sized in-memory buffers.
3. **TINY_BUFFER** — small buffers (worst to evict; usually kept longer).

If there is truly no memory available, DuckDB can **spill to disk** — it writes
blocks to a temporary file and reloads them later. This is handled by
`temporary_file_manager.cpp`.

### `buffer/block_handle.cpp` — A single block in memory

Each loaded block is represented by a `BlockHandle` object which tracks:
- The block's data in memory.
- How many parts of the code currently have it pinned.
- A sequence number for the eviction queue (newer = less likely to evict).
- Whether the block has been modified and needs to be written to disk.

### `buffer/buffer_handle.cpp` — A safe access wrapper

`BufferHandle` is like a "checkout receipt" — it proves you have pinned a block
and gives you access to its bytes. When the `BufferHandle` is destroyed (goes
out of scope), it automatically unpins the block.

---

## 5. `data_table.cpp` — The Actual Table

**File size:** ~73 KB  
**What it does:** This is the main object representing a table (like your `orders`
or `customers` table). All insert, update, delete, and scan operations on a table
go through this file.

### INSERT

When you insert rows:
1. New rows are first written to **local storage** (`local_storage.cpp`) — a
   transaction-private buffer. Other transactions cannot see these rows yet.
2. When you commit the transaction, the local storage is **merged** into the
   shared `RowGroupCollection`.
3. If you roll back, the local storage is simply discarded.

### SELECT (scan)

When you read rows:
1. DuckDB identifies which **row groups** might contain matching rows (using
   statistics — more on this later).
2. For each relevant row group, it reads the necessary **columns** (not all
   columns — only what the query needs).
3. It applies any **filters** as early as possible to skip rows without reading
   all their data.
4. It checks **transaction visibility** — rows deleted or inserted by other
   uncommitted transactions are hidden.

### DELETE and UPDATE

- **DELETE**: Does not immediately remove data. Instead, it marks rows as
  deleted in a **delete vector** (a bitmask where 1 = deleted). The data
  physically remains until the next checkpoint.
- **UPDATE**: Treated as a delete of the old row plus an insert of the new row.
  The old version is kept temporarily for transaction isolation.

---

## 6. `table/row_group_collection.cpp` — Managing Groups of Rows

**File size:** ~83 KB (the most complex single file)  
**What it does:** Tables are divided into chunks called **row groups**, each
containing up to 122,880 rows by default. This file manages the collection of
all row groups for one table.

### Why row groups?

Splitting a table into fixed-size row groups gives several advantages:
- Each row group can be **scanned independently and in parallel** — four CPU
  cores can each scan one row group simultaneously.
- **Statistics** (min value, max value, whether NULLs exist) are tracked
  per row group. If a query asks for `age > 90`, DuckDB can skip an entire
  row group if its max age is 85.
- **Checkpointing** can write one row group at a time, so large tables do not
  need to be flushed all at once.

### Key responsibilities

- Adding new row groups when the current one fills up.
- Assigning **row IDs** — a unique number for each row in the entire table.
- Merging locally-written rows (from transactions) into the persistent row groups.
- Supporting **reordering** — `row_group_reorderer.cpp` can physically re-sort
  row groups for better compression or scan performance.

---

## 7. `table/row_group.cpp` — One Chunk of ~120,000 Rows

**File size:** ~61 KB  
**What it does:** A single row group holds one column-per-data-column of the
table. It is the fundamental unit of DuckDB's columnar storage.

### Structure inside a row group

For a table with columns `(name, age, city, salary)`, a row group holds:

```
Row Group
  ├── Column 0: name   → [ "Alice", "Bob", "Carol", ... ]   (up to 122,880 values)
  ├── Column 1: age    → [ 30, 25, 35, ... ]
  ├── Column 2: city   → [ "New York", "Chicago", ... ]
  └── Column 3: salary → [ 50000, 45000, 60000, ... ]
```

Each column in the row group is a `ColumnData` object.

### Lazy loading

Columns are **not** loaded from disk until they are actually needed. If your
query only asks for `age` and `salary`, the `name` and `city` columns are never
loaded. This is a huge win for wide tables (hundreds of columns).

### Delete tracking

Each row group has a **version manager** (`row_version_manager.cpp`) that tracks
which rows have been deleted and by which transaction. This enables:
- Hiding deleted rows from new queries.
- Making deleted rows visible again if a transaction rolls back.

---

## 8. `table/column_data.cpp` — One Column's Data

**File size:** ~50 KB  
**What it does:** `ColumnData` manages all the data for a single column across
multiple **segments**. Think of it as a linked list of compressed data blocks.

### Segments

A column is divided into **segments** — each segment holds a portion of the
column's values, compressed using one algorithm. When one segment fills up,
a new segment is started.

```
Column "age"
  ├── Segment 1: rows 0–2047        compressed with bitpacking
  ├── Segment 2: rows 2048–4095     compressed with RLE (many repeated values)
  ├── Segment 3: rows 4096–6143     compressed with bitpacking
  └── ...
```

Different segments can use different compression algorithms — DuckDB picks the
best algorithm for each batch of data.

### Filter pushdown

If a query has `WHERE age > 30`, `ColumnData` can skip entire segments without
decompressing them by checking segment statistics:
- Segment 2 has max value 25? Skip it entirely — no row could match.
- Segment 5 has min 20, max 90? Must decompress and check each row.

### Specialized column types

Different data types have their own column implementations:

| File | Handles |
|------|---------|
| `standard_column_data.cpp` | Integers, floats, dates, timestamps |
| `list_column_data.cpp` | Lists/arrays like `[1, 2, 3]` |
| `struct_column_data.cpp` | Structs like `{name: "Alice", age: 30}` |
| `array_column_data.cpp` | Fixed-length arrays |
| `validity_column_data.cpp` | NULL tracking (which rows are NULL) |
| `variant_column_data.cpp` | JSON / flexible-type data |
| `geo_column_data.cpp` | Geometry / spatial data |

---

## 9. `table/column_segment.cpp` — A Compressed Data Block

**What it does:** A `ColumnSegment` is the lowest-level storage unit — it is a
single compressed block of values for one column. It knows:
- Which compression algorithm was used.
- How many values are in this segment.
- How to scan (read) values out.
- How to append new values.

When scanning, the segment **decompresses on the fly** directly into the
vectorized execution engine's batch buffer, producing 2,048 values at a time.

---

## 10. `compression/` — The Compression Library

DuckDB uses many different compression algorithms and picks the best one for
each column segment automatically. Here is what each does:

### `bitpacking.cpp` (~39 KB) — Bit-level integer packing

**What it does:** Integers take 64 bits of space by default, but if all values
in a column are between 0 and 100, they only need 7 bits each. Bitpacking
stores each integer using only as many bits as needed.

**Example:**  
Values: `[3, 7, 1, 5, 4]` — all fit in 3 bits  
Stored as: `011 111 001 101 100` (15 bits total instead of 5 × 64 = 320 bits)

**Variants:**
- **Frame-of-reference**: Subtracts the minimum value first. If values are
  `[1000, 1001, 1002, 1003]`, subtract 1000 to get `[0, 1, 2, 3]` — now only
  2 bits each are needed.
- **Delta encoding**: Stores the *difference* between consecutive values.
  `[100, 102, 105, 110]` → `[100, +2, +3, +5]`. Great for sorted data.
- **Constant detection**: If all values are the same, store just one value and
  a count. Takes almost no space.

### `rle.cpp` (~23 KB) — Run-Length Encoding

**What it does:** If a column has many consecutive repeated values (e.g., a
`status` column where thousands of rows are `"active"`), store `("active", 5000)`
instead of repeating `"active"` 5,000 times.

**Example:**  
Input: `[A, A, A, A, B, B, C, C, C]`  
RLE output: `[(A, 4), (B, 2), (C, 3)]` — 3 pairs instead of 9 values.

### `dictionary_compression.cpp` (~9 KB) — Dictionary Encoding

**What it does:** For columns with many repeated string values (like a `country`
column with only 200 distinct countries across millions of rows), build a lookup
table:

```
Dictionary: { 0: "USA", 1: "Canada", 2: "Mexico", ... }
Data:        [ 0, 0, 1, 2, 0, 0, 0, 1, ... ]
```

Instead of storing the full string for every row, store a small integer index.

### `fsst.cpp` (~36 KB) — Fast Static Symbol Table

**What it does:** A more advanced string compression. It analyses the strings in
a column, finds the most common substrings (like "http://", ".com", "2024-"),
and replaces each with a single byte code. Good for URLs, email addresses, log
lines.

### `zstd.cpp` (~42 KB) — Zstandard (General-Purpose Compression)

**What it does:** A well-known compression algorithm (also used in Linux kernels,
Facebook's servers, etc.) that works by finding repeated patterns of any kind.
It is slower than bitpacking but can compress arbitrary data very well.

### `alp/` — Adaptive Lossless Float Compression

**What it does:** Floating-point numbers (like `3.14159`) are tricky to compress
because their bit patterns look random. ALP finds a way to represent floats as
integers internally (without losing precision) and then bitpacks those integers.
Typically 4–8× compression on scientific data.

### `chimp/` — CHIMP Float Compression

**What it does:** Another float compression algorithm designed for time-series
data (e.g., temperature readings, sensor data) where consecutive values are
similar. Stores only the *differences* between consecutive float values using
a compact bit encoding.

### `roaring/` — Roaring Bitmaps

**What it does:** A highly efficient data structure for sets of integers (often
used for NULL tracking and delete vectors). Roaring bitmaps adaptively use
different representations depending on density:
- Sparse sets (few 1s): store as a sorted list.
- Dense sets (many 1s): store as a raw bitmap.
- Very dense sets: store as a run-length encoded list of ranges.

### How compression is chosen

DuckDB **automatically picks the best algorithm** for each segment. When filling
a new segment, it runs the incoming data through a **sampling process** — it
checks which algorithm would produce the smallest output and uses that one. The
winner is recorded in the segment's metadata so it can be decompressed later.

---

## 11. `table/update_segment.cpp` — Tracking Changes (MVCC)

**File size:** ~53 KB  
**What it does:** This is the heart of DuckDB's transaction isolation system.
When rows are updated or deleted, the old versions must be kept temporarily so
that:
- Transactions that started before the change can still see the old data.
- If a transaction rolls back, the old data is restored.

### The delete vector

Every row group has a **delete vector** — a bitmap where each bit corresponds
to one row:
- `0` = row is alive
- `1` = row has been deleted

When a transaction deletes rows, it flips bits in the delete vector. Other
transactions check this vector when scanning to skip deleted rows. When the
deleting transaction commits, the deletion becomes permanent. If it rolls back,
the bits are flipped back.

### Update chains

When a row is updated, DuckDB creates a **version chain**:

```
Current version → Old version v2 → Old version v1 → (original)
```

Each version knows which transaction created it. When reading, DuckDB walks
the chain to find the correct version for the current transaction's snapshot.

---

## 12. `table/chunk_info.cpp` — Visibility Metadata

**File size:** ~17 KB  
**What it does:** Each data chunk (2,048 rows) has a `ChunkInfo` that records
the **transaction IDs** of insertions and deletions. When scanning, DuckDB
uses this to quickly determine whether any row in a chunk could be visible to
the current transaction, or whether the entire chunk can be skipped.

Three types of chunk info:
- `ChunkConstantInfo` — all rows have the same visibility (optimized for bulk
  inserts — very common at checkpoint time).
- `ChunkVectorInfo` — each row has individual visibility tracking (for chunks
  with mixed transaction history).

---

## 13. `statistics/` — The Query Accelerator

**What it does:** Every column segment, row group, and table maintains
**statistics** — metadata that describes the data without looking at every row.
These statistics allow the optimizer and executor to skip huge amounts of data.

### Types of statistics

| File | Tracks |
|------|--------|
| `numeric_stats.cpp` | Min value, max value, whether NULLs exist |
| `string_stats.cpp` | Shortest/longest string, min/max string, has NULLs |
| `distinct_statistics.cpp` | Approximate count of unique values (HyperLogLog) |
| `struct_stats.cpp` | Statistics for each field of a struct |
| `list_stats.cpp` | Statistics for elements within lists |
| `geometry_stats.cpp` | Bounding box for spatial data |

### How they speed up queries

**Zone maps (min/max skipping):**  
For a query `WHERE salary > 200000`:
- Row group 3 has max salary = 95,000 → skip the entire row group.
- Row group 7 has min salary = 150,000, max = 300,000 → must scan it.

This is called a "zone map" and can skip 90%+ of data for selective queries on
sorted or partially-sorted columns.

**NULL skipping:**  
For `WHERE email IS NOT NULL`, skip any segment that has no NULLs entirely
(no need to even check the NULL bitmap).

---

## 14. `checkpoint_manager.cpp` and `checkpoint/` — Saving to Disk

**What it does:** A **checkpoint** is a consistent, complete snapshot of the
entire database written to the main `.db` file. After a successful checkpoint,
the WAL up to that point can be discarded.

### When does a checkpoint happen?

- Automatically when the WAL grows beyond a configured size (default: 16 MB).
- Manually when you call `CHECKPOINT;` in SQL.
- When the database is closed cleanly.

### The checkpoint process (`checkpoint_manager.cpp`, ~29 KB)

1. **Acquire a checkpoint lock** — prevents new write transactions from starting
   during the checkpoint.
2. **Wait for active transactions to finish** — ensures a consistent snapshot.
3. **For each table:** call `table_data_writer.cpp` to serialize all row groups
   to disk.
4. **Write catalog metadata** — table names, column definitions, constraints, etc.
5. **Write the new root block** — a special block at the start of the file that
   says "the valid checkpoint starts here."
6. **Sync the file** — forces the OS to flush all data to physical disk.
7. **Truncate the WAL** — the log is no longer needed up to this point.

### `checkpoint/table_data_writer.cpp` (~8 KB)

Writes one table's worth of data:
1. Scans the table row group by row group.
2. For each row group, serializes each column's segments.
3. Runs in parallel using a `TaskExecutor` so multiple row groups are written
   simultaneously.

### `checkpoint/write_overflow_strings_to_disk.cpp` (~5 KB)

Strings longer than ~12 bytes cannot fit inside a column segment block.
They are stored in separate **overflow blocks** and the column stores a
pointer (block ID + offset) to find them. This file handles allocating those
overflow blocks and linking them together with a chain of block IDs.

### `checkpoint/row_group_writer.cpp` (~2 KB)

Serializes a single row group's metadata (which blocks contain its columns,
compression info, row count, statistics).

---

## 15. `local_storage.cpp` — Your Private Scratch Pad

**File size:** ~30 KB  
**What it does:** When a transaction inserts or modifies data, those changes
are kept private in `LocalStorage` — invisible to all other transactions — until
the transaction commits.

### Why not write directly to the shared table?

If you write directly to the shared table and then roll back, you would need to
undo all those changes. Instead, DuckDB keeps a separate private copy and only
merges it into the shared table at commit time.

### Structure

Local storage mirrors the structure of the real table — it also has row groups
and columns — but it lives entirely in memory (or spills to temp files if large).

At commit time, the local row groups are appended to the shared `RowGroupCollection`.

---

## 16. `temporary_file_manager.cpp` and `temporary_memory_manager.cpp` — Spilling to Disk

**What they do:** When a query needs more memory than is available (e.g., a huge
sort or hash join), DuckDB writes temporary data to disk files and reads it back
as needed. This is called **spilling**.

- `temporary_memory_manager.cpp` tracks how much temporary memory is being used
  by all operations.
- `temporary_file_manager.cpp` manages the actual temporary files on disk:
  allocating space, writing buffers, reading them back, and cleaning up when done.

Temporary files are separate from the main `.db` file and are automatically
deleted when the query finishes or the database is closed.

---

## 17. `metadata/` — The Database's Table of Contents

**What it does:** The **metadata blocks** are special blocks at the start of the
database file that describe its structure — where each table's data lives, what
the column names and types are, which indexes exist, etc.

- `metadata_manager.cpp` — allocates and tracks metadata blocks.
- `metadata_writer.cpp` — serializes metadata objects to binary format.
- `metadata_reader.cpp` — reads metadata back and reconstructs the objects.

Without these, DuckDB would not know where anything is in the file.

---

## 18. `external_file_cache/` — Caching Remote Files

**What it does:** DuckDB can query files that are not on local disk — such as
files stored in Amazon S3 or Azure Blob Storage. Reading every needed byte
over the network would be very slow. The external file cache stores recently
accessed portions of remote files locally so they can be reused without
re-downloading.

- `caching_file_system.cpp` — wraps the network file system with a caching layer.
- `external_file_cache_block.cpp` — one cached chunk of a remote file.
- `file_buffer_handle_group.cpp` — groups related file buffers together.

---

## 19. `serialization/` — Saving Non-Data Objects

**What it does:** Not everything in the database is raw row data. Table
definitions, column types, constraints, indexes, and query-plan fragments also
need to be saved to disk and reloaded. The serialization folder handles
converting these C++ objects into binary bytes and back.

- `serialize_storage.cpp` — top-level storage structures.
- `serialize_parsed_expression.cpp` — saves the expressions in default values,
  check constraints (`age > 0`), etc.
- `serialize_tableref.cpp` — saves references to tables (used in views).
- `serialize_constraint.cpp` — saves `NOT NULL`, `UNIQUE`, `PRIMARY KEY`, etc.

---

## 20. `arena_allocator.cpp` — Fast Throwaway Memory

**What it does:** Many operations need to allocate lots of small objects very
quickly and then throw them all away at once (e.g., building a hash table during
a join). An **arena allocator** works like a notepad:

- You allocate as much as you want by just moving a pointer forward.
- You never free individual items.
- When you are done, you tear out the whole page — deallocating everything in
  one instant.

This is much faster than allocating and freeing each small object individually.

---

## End-to-End Example: What Happens When You Insert a Row

Let's trace `INSERT INTO orders VALUES (101, 'Widget', 29.99)` through the
storage engine:

1. **Transaction starts** — a transaction ID is assigned.

2. **`data_table.cpp`** — receives the insert request.

3. **`local_storage.cpp`** — the new row is written to the transaction's private
   local storage. Still invisible to others.

4. **`write_ahead_log.cpp`** — a `WAL_INSERT_TUPLE` record is written to the
   WAL file. If the computer crashes before commit, this ensures the insert
   can be recovered.

5. **`COMMIT`** — the transaction commits.

6. **`local_storage.cpp`** — the local row is merged into the shared
   `RowGroupCollection`.

7. **`row_group_collection.cpp`** — finds the current row group (or creates a
   new one if the last is full) and appends the row.

8. **`column_data.cpp`** — each column value is appended to its respective
   column segment.

9. **`column_segment.cpp`** — if the segment is full, a new one is started.
   The compression algorithm picks the best algorithm for the new batch.

10. **Statistics updated** — the row group's min/max statistics are updated
    for each column.

The data is now in memory. It will be persisted to the `.db` file on the next
checkpoint.

---

## End-to-End Example: What Happens When You Query Rows

`SELECT SUM(price) FROM orders WHERE customer_id = 42`:

1. **`data_table.cpp`** receives the scan request with filter `customer_id = 42`.

2. **`row_group_collection.cpp`** iterates over all row groups.

3. **Statistics check** — for each row group, check: does this row group's
   `customer_id` min–max range include 42? If not, skip the entire row group.

4. **`row_group.cpp`** — for surviving row groups, only load the
   `customer_id` and `price` columns. The rest are never touched.

5. **`column_data.cpp`** — for each column segment:
   - Check segment statistics (min/max `customer_id`). Skip if 42 is out of range.
   - Decompress the segment into a batch of 2,048 values.

6. **`chunk_info.cpp`** — check the delete vector to skip deleted rows.

7. **Transaction visibility** — skip rows inserted by uncommitted transactions.

8. **Filter applied** — keep only rows where `customer_id = 42`.

9. **`price` values summed** — in the vectorized execution engine (outside
   storage, in `src/execution/`).

10. **Result returned** to the user.

---

## Summary Table

| Component | File(s) | Job |
|-----------|---------|-----|
| Supervisor | `storage_manager.cpp` | Opens/creates database, sets everything up |
| Safety log | `write_ahead_log.cpp`, `wal_replay.cpp` | Write-before-change, crash recovery |
| Block manager | `single_file_block_manager.cpp` | Assigns block IDs, manages the .db file |
| Memory manager | `standard_buffer_manager.cpp`, `buffer/buffer_pool.cpp` | Keeps hot data in RAM, evicts cold data |
| Table | `data_table.cpp` | Handles insert/update/delete/scan |
| Row group collection | `table/row_group_collection.cpp` | Manages all row groups for a table |
| Row group | `table/row_group.cpp` | One chunk of ~120,000 rows |
| Column data | `table/column_data.cpp` | One column's data across segments |
| Column segment | `table/column_segment.cpp` | One compressed block of values |
| Compression | `compression/*.cpp` | Packs data tightly to save space |
| MVCC / versions | `table/update_segment.cpp`, `table/chunk_info.cpp` | Makes transactions see correct data |
| Statistics | `statistics/*.cpp` | Min/max/null info for skipping data |
| Checkpoint | `checkpoint_manager.cpp`, `checkpoint/*.cpp` | Saves snapshot to disk |
| Local storage | `local_storage.cpp` | Private scratch pad for each transaction |
| Temp spill | `temporary_file_manager.cpp` | Overflow to disk when memory runs out |
| Metadata | `metadata/*.cpp` | Table of contents for the .db file |
| Serialization | `serialization/*.cpp` | Saves non-row objects (schemas, constraints) |
| Arena allocator | `arena_allocator.cpp` | Fast bulk memory allocation |
