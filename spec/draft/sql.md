- **`schema.sql`** — a single file containing all DDL. Loaded first.
- **`data/`** — a folder containing one file per source table with `INSERT` statements. Loaded after `schema.sql`, in any order (foreign keys are respected because DDL creates constraints deferrable).

This structure exists so consumers can:

- Load the schema separately and inspect it before loading data
- Load only the tables they need
- Parallelise data loading across tables where their infrastructure supports it
- Skip failed table loads without losing the whole export

## Encoding and Formatting

- **UTF-8 without BOM.** Consistent with CSV and JSON.
- **LF (`\n`)** line endings.
- Each SQL statement ends with a semicolon on its own line.
- Blank line between statements for readability.
- Comments (Postgres `-- ...` style) at the top of every file identifying the source table and generation time.

## Schema File (`schema.sql`)

The `schema.sql` file contains all DDL required to recreate the database structure. Loaded first, it establishes tables, columns, constraints, and indexes.

### Order of statements

Statements are written in a strict order to satisfy dependency requirements:

1. Schema comment header (metadata about the export)
2. `CREATE TABLE` statements — all tables, alphabetically by name
3. `ALTER TABLE ... ADD PRIMARY KEY` statements
4. `ALTER TABLE ... ADD CONSTRAINT ... FOREIGN KEY` statements — marked as `DEFERRABLE INITIALLY DEFERRED`
5. `CREATE INDEX` statements

Foreign keys are added *after* all tables exist, and marked as deferrable so that data loading can happen in any table order without integrity check failures during the load.

### What's included

- **Tables** — column names, Postgres types, `NOT NULL` constraints, `DEFAULT` values where present in source
- **Primary keys** — as `ALTER TABLE ... ADD PRIMARY KEY` statements
- **Foreign keys** — as deferrable constraints referencing the correct tables and columns
- **Indexes** — non-primary-key indexes present on the source, including partial indexes and expression indexes where applicable

### What's excluded from v1.0

- **Views** — Postgres-specific and rarely needed for archived data
- **Sequences** — auto-increment behaviour is source-specific; consumers wanting sequences can add them
- **Triggers** — behavioural code that shouldn't run on archived data
- **Stored procedures** — same rationale
- **Row-level security policies** — consumer's environment will have its own security model
- **Permissions and grants** — consumer's environment has its own users
- **Extensions** — consumer's Postgres instance has its own installed extensions

Consumers needing any of these can request them as a customisation, but v1.0 keeps the schema focused on what's needed to reload the *data*, not to recreate the *source system*.

### Example schema.sql

```sql
-- Archiva Format Export
-- Source: SynergySoft (MultiValue / UniVerse)
-- Generated: 2026-08-15T14:30:00Z
-- Format version: 1.0

-- Tables

CREATE TABLE customers (
    id BIGINT NOT NULL,
    name TEXT NOT NULL,
    balance NUMERIC(12, 2) NOT NULL DEFAULT 0.00,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL
);

CREATE TABLE invoices (
    id BIGINT NOT NULL,
    customer_id BIGINT NOT NULL,
    amount NUMERIC(12, 2) NOT NULL,
    issued_at DATE NOT NULL
);

-- Primary keys

ALTER TABLE customers ADD PRIMARY KEY (id);
ALTER TABLE invoices ADD PRIMARY KEY (id);

-- Foreign keys

ALTER TABLE invoices
    ADD CONSTRAINT invoices_customer_id_fkey
    FOREIGN KEY (customer_id) REFERENCES customers (id)
    DEFERRABLE INITIALLY DEFERRED;

-- Indexes

CREATE INDEX invoices_customer_id_idx ON invoices (customer_id);
```

## Data Files (`data/*.sql`)

Each source table produces one file in `data/`, named after the table.

### Statement style

Data is written as **multi-row INSERT statements**. Each statement inserts up to **1000 rows**.

```sql
INSERT INTO customers (id, name, balance, created_at) VALUES
    (1, 'Jane Cooper', 1250.00, '2024-08-15T09:30:00+00:00'),
    (2, 'John Smith', 890.50, '2024-08-16T14:15:00+00:00'),
    ...
    (1000, 'Robert Wilson', 2100.75, '2024-09-01T11:00:00+00:00');
```

**Why multi-row:** dramatically faster to load than single-row. Standard practice for SQL dumps.

**Why 1000 rows per statement:** a balanced batch size — small enough that individual statement failures don't lose too much progress, large enough for good load performance. Configurable in v1.1 if customers demonstrate need.

### Transactional wrapping

Each data file is wrapped in a **single transaction**:

```sql
BEGIN;

INSERT INTO customers ... ;
INSERT INTO customers ... ;
INSERT INTO customers ... ;

COMMIT;
```

Per-table transactions mean:

- One broken table doesn't roll back others
- Consumers get clear failure boundaries
- Partial-load recovery is straightforward
- Foreign keys deferrable at schema level means order-independent loading

### Row ordering

- Rows are written in **primary key order** for reproducibility
- Same source snapshot produces byte-identical output

### Column order

- Columns are listed in `INSERT` statements in **source ordinal position** — the column order defined in `CREATE TABLE`
- Not alphabetical, not primary-key-first

## Data Type Representation

SQL preserves types natively via DDL declarations. Data values are written with these conventions:

### NULL

- Written as the SQL literal `NULL` (unquoted, uppercase)
- Not `''` (empty string), which would insert an empty string instead of NULL

### Booleans

- Written as `TRUE` / `FALSE` (uppercase, SQL literal)
- Not `1` / `0` — Postgres accepts both but boolean columns should carry boolean literals

### Integers

- Written as decimal digits, no quotes
- Example: `42`, `-17`, `1000000`

### Decimals

- Written as decimal literals, no quotes
- Precision preserved to match source declaration
- Example: `1250.00`, `1250.007500`

### Text (strings)

- Wrapped in **single quotes**
- Embedded single quotes escaped by doubling: `'O''Brien'`
- Backslash escaping **not** used
- Example: `'Jane Cooper'`, `'123 Main St, Apt ''B'''`

### Dates

- Wrapped in single quotes, ISO 8601 format
- Example: `'2024-08-15'`

### Timestamps

- Wrapped in single quotes, ISO 8601 format
- Example: `'2024-08-15T14:30:00'` (without timezone)
- Example: `'2024-08-15T14:30:00+10:00'` or `'2024-08-15T14:30:00Z'` (with timezone)

### JSON columns

- Wrapped in single quotes, containing valid minified JSON
- Postgres will cast the string to `jsonb` on insert if the target column is `jsonb`
- Example: `'{"key":"value"}'`

### Arrays

- Written using Postgres array literal syntax
- Example: `ARRAY['a', 'b', 'c']` for text arrays; `ARRAY[1, 2, 3]` for integer arrays
- Preserves native Postgres array semantics

### Binary data

- Written as `decode('base64_string', 'base64')`
- Example: `decode('SGVsbG8gd29ybGQ=', 'base64')`

## Archive Packaging

- All SQL files are packaged into the archive alongside `manifest.json`
- Structure at archive root:
  manifest.json
  sql/
  schema.sql
  data/
  customers.sql
  invoices.sql
  ...

  - If other formats are also included, they sit alongside in `csv/`, `json/`, `parquet/` folders

## Producer Behaviour

An Archiva Format producer generating SQL:

1. **Reads the schema** from Postgres for each entitled table
2. **Writes** `schema.sql` with all DDL in the correct dependency order
3. **For each table**, opens a data file and writes a `BEGIN;`
4. **Streams rows** in primary key order from Postgres
5. **Batches** rows into multi-row INSERT statements (1000 rows per statement)
6. **Writes** each batch as one INSERT statement
7. **Writes** `COMMIT;` at the end of the data file
8. **Computes SHA-256** of each file
9. **Records** each file in the manifest with row count (data files), byte count, and checksum
10. **Includes** all files in the archive

Reproducibility requirement: given the same source snapshot, two runs must produce byte-identical `schema.sql` and byte-identical data files.

## Consumer Guidance

For customers or their engineers loading an SQL export:

- **Load `schema.sql` first.** This creates all tables, primary keys, foreign keys, and indexes.
- **Then load data files.** Order does not matter — foreign keys are deferrable.
- **Each data file is atomic.** A failed data file rolls back that table's inserts, but leaves other tables intact.
- **Verify checksums** against the manifest before loading.
- **Postgres is the target dialect.** For other databases:
  - Identifier quoting (`"..."` vs backticks)
  - Type names (`TEXT` vs `NVARCHAR(MAX)`, `NUMERIC` vs `DECIMAL`)
  - Timestamp with timezone syntax
  - JSON column handling
  - Array types (may need normalisation into related tables)
- **Consult the manifest** for column types before writing loading logic.

## Non-Goals for v1.0

The following are deliberately not addressed by v1.0 of this specification:

- **Non-Postgres dialects.** Planned for v1.1 as a customer-selectable option.
- **Views, sequences, triggers, stored procedures.** Source-system behavioural code that shouldn't run on archived data.
- **Extensions, permissions, row-level security.** Consumer environment concerns.
- **`COPY FROM` fast load format.** Postgres-native and dramatically faster; planned for v1.1 as an optional variant.
- **Compressed SQL files.** Compression is handled at the archive level.
- **Configurable batch size.** 1000 rows per statement is fixed in v1.0.

## Rationale for Specific Choices

**Why Postgres as the default dialect:** Archiva's pipeline is Postgres-native. Exporting anything else would be lossy translation. Postgres is also the most expressive dialect, so information isn't lost.

**Why `schema.sql` + `data/*.sql` split:** loading order matters. Splitting lets consumers control what loads when, retry failed tables independently, and inspect the schema before touching data.

**Why deferrable foreign keys:** table load order shouldn't matter. Deferrable constraints let consumers load data files in parallel or any order without integrity check failures during the load.

**Why per-table transactions:** balances safety and progress. A single mega-transaction rolls back everything on any failure. No transactions leaves partial state. Per-table gives clear boundaries.

**Why multi-row INSERTs at 1000 rows per statement:** single-row INSERTs are dramatically slower to load. 1000 rows is a reasonable batch — large enough for good performance, small enough that individual statement failures don't destroy too much progress.

**Why include indexes and foreign keys but not views:** indexes and foreign keys are integrity information — they describe what the data means. Views are derived computation — they belong to the application layer, not the archive layer.

**Why not include triggers, procedures, sequences:** these are behavioural. Archived data shouldn't trigger behavioural code — the actions those triggers represented already happened in the source system. Including them risks accidental side effects on consumer load.

**Why `DEFERRABLE INITIALLY DEFERRED`:** deferred foreign key checks happen at transaction commit, not statement execution. This lets us load data in any order within a transaction. The consumer's load process gets order-independence without sacrificing eventual integrity.

## Version History

- **Draft (v1.0)** — Initial specification. Establishes Postgres as the default dialect, `schema.sql` + `data/*.sql` split, multi-row INSERTs with per-table transactions, deferrable foreign keys.
