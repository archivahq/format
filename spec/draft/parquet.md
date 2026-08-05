# Parquet Format Specification

**Version:** Draft (for inclusion in v1.0)

## Purpose

Parquet is the export format for large-scale analytical use. Where CSV serves the widest audience of tools, JSON serves structured applications, and SQL serves database rebuild, **Parquet serves analytics and data teams.**

A Parquet export is dramatically smaller than the equivalent CSV, dramatically faster to load into modern analytical tools, and preserves types natively — no manifest reference required at read time to know that a column is a date, a decimal, or a boolean. It is what a competent data team will ask for once they see the alternative.

## What Parquet is best for

- Large historical tables — a decade of transactions, a full ratepayer register
- Analytics workloads — loading into Power BI, Tableau, DuckDB, cloud data warehouses
- Long-term storage where efficient querying matters
- Preserving native types through export — Parquet carries its own schema

## What Parquet is worst for

- Human review at the row level — Parquet files are binary; a text editor shows nothing useful
- Direct import into a spreadsheet — requires a converter or plugin
- Small datasets — the format's overhead is only worth it above a few thousand rows
- Environments without modern data tooling — a records officer with only Excel cannot open Parquet directly

Customers with these needs are directed to CSV, JSON, or SQL.

## File Layout

### One file per source table

Each Postgres table produces one `.parquet` file. Files are named after the table, in lowercase:
customers.parquet
invoices.parquet
invoice_lines.parquet

Files are not merged, joined, or denormalised. The customer receives the schema as it was at extraction, with foreign key relationships preserved through matching column values.

### Row groups

Parquet organises data into **row groups** — chunks of rows that are compressed and read together. Row group size affects the balance between compression efficiency and random-access speed.

**Archiva's convention: 128 MB target row group size.**

This is Parquet's own historical default and works well with every major reader. Larger row groups compress marginally better but slow random-access queries; smaller row groups load faster but compress less efficiently. 128 MB is the well-understood middle.

Tables smaller than 128 MB produce a single row group. Tables larger produce multiple row groups sized to approximately 128 MB each.

### Compression

**All Parquet files use Zstandard (Zstd) compression at the column level.**

Zstd is the modern default in the Parquet ecosystem — excellent compression ratio (typically 3–5x smaller than uncompressed for typical ERP data), fast decompression (comparable to Snappy for practical read speeds), and universal support in every current Parquet reader.

Older alternatives (Snappy, Gzip) are not used. Zstd supersedes both on ratio and modern tooling handles it natively.

## Schema Encoding

Parquet embeds its schema in every file. Consumers reading a Parquet file receive column names, types, and nullability without needing to consult the manifest.

### Column names

Column names in the Parquet schema match the source Postgres column names exactly, including case. No renaming, no case conversion, no prefixing.

### Types

Postgres types are mapped to Parquet's logical types with faithful precision:

| Postgres type | Parquet logical type |
| --- | --- |
| `boolean` | `BOOLEAN` |
| `integer` | `INT32` |
| `bigint` | `INT64` |
| `smallint` | `INT32` (with `INTEGER(16, true)` annotation) |
| `numeric(p, s)` | `DECIMAL(p, s)` — stored as `FIXED_LEN_BYTE_ARRAY` |
| `real` | `FLOAT` |
| `double precision` | `DOUBLE` |
| `text`, `varchar`, `char` | `BYTE_ARRAY` with `STRING` logical type (UTF-8) |
| `date` | `INT32` with `DATE` logical type |
| `time` | `INT64` with `TIME_MICROS` logical type |
| `timestamp` | `INT64` with `TIMESTAMP_MICROS` logical type, UTC-adjusted false |
| `timestamp with time zone` | `INT64` with `TIMESTAMP_MICROS` logical type, UTC-adjusted true |
| `jsonb`, `json` | `BYTE_ARRAY` with `JSON` logical type |
| Postgres arrays | `LIST` with the element type nested |
| `bytea` | `BYTE_ARRAY` with no string annotation |

### Nullability

Each column's nullability is encoded in the schema:

- Nullable columns are marked `OPTIONAL`
- Non-nullable columns are marked `REQUIRED`

Consumers receive nullability information natively from the file; the manifest carries the same information for cross-format consistency.

### Decimals

Decimals are stored as `DECIMAL(precision, scale)` using Parquet's `FIXED_LEN_BYTE_ARRAY` physical type — the canonical Parquet representation. Precision and scale match the source Postgres declaration exactly. No precision is lost.

Consumers reading with a Parquet reader receive the decimal as a decimal, not as a floating-point number. This is Parquet's structural advantage over JSON for financial data — no string conversion required, no precision compromise.

### Timestamps

Postgres `timestamp` (without timezone) is stored as `TIMESTAMP_MICROS` with `isAdjustedToUTC = false`.
Postgres `timestamp with time zone` is stored as `TIMESTAMP_MICROS` with `isAdjustedToUTC = true`.

Microsecond precision is preserved. Consumers get correct local-vs-UTC semantics from the schema.

### JSON and JSONB

Postgres `json` and `jsonb` columns are stored as UTF-8 `BYTE_ARRAY` with the `JSON` logical type annotation. Consumers with JSON-aware readers can parse the values directly.

### Arrays

Postgres array columns are stored using Parquet's native `LIST` type with the element type nested. A `text[]` column becomes a `LIST<STRING>`. An `integer[]` column becomes a `LIST<INT32>`. Consumers receive arrays as arrays, not as stringified representations.

Multi-dimensional Postgres arrays are supported but flagged in the manifest — some Parquet readers handle nested lists less cleanly, and the customer should be aware.

## Statistics

Parquet supports per-column statistics stored at the row group level: minimum value, maximum value, null count, and distinct count.

**Archiva's convention: all statistics are computed and included.**

Statistics dramatically improve query performance for analytical workloads — a reader can skip entire row groups if it can prove they cannot contain matching data. There is no meaningful cost to producing statistics at export time, and consumers benefit directly.

## Row Ordering

Rows within each file are written in **primary key order**. Same as CSV, JSON, and SQL. This is a reproducibility requirement — regenerating the same export from the same snapshot must produce byte-identical files.

Where a table has no primary key (rare in Archiva engagements but occasionally seen in staging or lookup tables), rows are written in the source system's natural insertion order, and the manifest flags the absence of a primary key.

## Archive Packaging

Parquet files for an engagement are packaged into the archive alongside `manifest.json`:
manifest.json
parquet/
customers.parquet
invoices.parquet
invoice_lines.parquet
...

Compression is at the column level within each Parquet file. **The archive itself is not additionally compressed** — Zstd within Parquet is already efficient, and re-compressing at the zip level wastes CPU with no meaningful benefit. Zip is used for archive packaging only, not for further compression.

## Producer Behaviour

An Archiva Format producer generating Parquet:

1. **Reads the schema** for each entitled table from Postgres, including precision and scale for numerics
2. **Constructs** the Parquet schema with faithful type mapping and nullability
3. **Streams rows** in primary key order
4. **Writes rows into row groups** sized to approximately 128 MB
5. **Applies Zstd compression** at the column level within each row group
6. **Computes column statistics** (min, max, null count, distinct count) for every column of every row group
7. **Finalises** the file with embedded schema and statistics
8. **Computes SHA-256** of the completed file
9. **Records** the file in the manifest with row count, byte count, and checksum
10. **Includes** the file in the final archive

Reproducibility requirement: given the same source snapshot and the same producer version, two independent runs must produce byte-identical Parquet files. This constrains implementation details — sort order, row group boundaries, and compression parameters must all be deterministic.

## Consumer Guidance

For customers or their engineers reading Parquet exports:

- **Any modern Parquet reader works.** DuckDB, pandas (via PyArrow), Spark, Trino, cloud data warehouses. All handle Zstd, all handle the Parquet 2.x specification, all handle the type mappings used here.
- **The schema is in the file.** No manifest reference needed to know column types — Parquet carries them natively.
- **Verify the checksum** against the manifest before loading. Parquet files are binary; corruption is undetectable by eye.
- **Statistics accelerate queries.** Filter on indexed columns (typically primary keys, dates, foreign key columns) to take advantage of row-group pruning.
- **Decimal columns arrive as decimals.** If your reader defaults to floating-point, override — Archiva preserves exact precision and it would be a shame to lose it at load time.
- **Timestamps carry timezone awareness.** Respect the `isAdjustedToUTC` flag on timestamp columns; misreading this can shift date-partitioned analytics by a day.

## Non-Goals for v1.0

The following are deliberately not addressed by v1.0:

- **Configurable compression algorithm.** Zstd is fixed. Alternative codecs (Snappy, Gzip, LZ4) are not offered — Zstd supersedes them for archival use.
- **Configurable row group size.** 128 MB is fixed. Tuning belongs in v1.1 if customers demonstrate need.
- **Dictionary encoding tuning.** Parquet's default dictionary encoding is used. Explicit encoding hints are not part of v1.0.
- **Encryption at the column level.** Parquet supports column-level encryption in recent versions; Archiva does not use it in v1.0. Sensitive-data handling belongs at the archive packaging level.
- **Multi-file datasets with partitioning.** Datasets partitioned by date or region across many files are a common Parquet pattern for very large data. Not used in v1.0 — one file per table is the convention.

## Rationale for Specific Choices

**Why Zstd:** the modern default. Better compression than Snappy for archival data, decompression speed close enough that analytical readers don't notice. Every current Parquet reader supports it. No good reason to pick anything else.

**Why 128 MB row groups:** Parquet's own default, well-understood, works with everything. Larger row groups compress better but slow random access; smaller groups load faster but compress less. 128 MB is where the mainstream tooling assumes.

**Why one file per table:** consistency with CSV, JSON, and SQL. Simpler mental model. Per-file checksums in the manifest give per-table integrity verification. Partitioned multi-file datasets are more efficient at very large scale but add complexity that most Archiva engagements don't need.

**Why all statistics on:** zero cost at write time, real benefit at read time. Modern analytical readers use statistics for row-group pruning, which can make a query 10x faster. Not producing them would be leaving performance on the table for no reason.

**Why native Parquet types instead of strings:** the whole point of Parquet over CSV is native type preservation. Decimals arrive as decimals, timestamps arrive as timestamps, dates arrive as dates. Downgrading any of these to strings would defeat the format's purpose.

**Why not compress the archive itself:** Parquet's column-level compression is already efficient. Zip-level compression on top would gain a few percent of space at real CPU cost. The archive uses zip for packaging structure only, not for further compression.

**Why microsecond timestamp precision:** matches Postgres native precision. Nanosecond precision (also supported by Parquet) offers no benefit for ERP data and reduces reader compatibility. Millisecond precision would lose information Postgres holds.

## Version History

- **Draft (v1.0)** — Initial specification. Establishes Zstd compression, 128 MB row groups, one file per table, statistics enabled, faithful Postgres type mapping.
