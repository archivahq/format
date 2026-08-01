# CSV Format Specification

**Version:** Draft (for inclusion in v1.0)

## Purpose

CSV is the universal export format. It exists to be readable by any tool, on any platform, in any decade — including tools that do not yet exist. Every Archiva engagement includes CSV as the baseline handover format unless explicitly excluded.

## What CSV is best for

- Finance ledgers, HR lists, ratepayer registers, and other tabular records
- Auditor review, records-office inspection, spreadsheet-based analysis
- Import into another system that reads CSV — which is essentially all of them
- Long-term archival where readability outweighs efficiency

## What CSV is worst for

- Hierarchical data with nested relationships
- Very large datasets where file size and read performance matter
- Data with rich typing that CSV cannot represent (every value is text)
- Round-tripping through systems that impose their own quoting or encoding assumptions

Customers with these needs are directed to Parquet, JSON, or SQL.

## File Layout

### One file per source table

Each Postgres table in the export produces one `.csv` file. Files are named after the table, in lowercase, with underscores preserved:
customers.csv
invoices.csv
invoice_lines.csv

Tables are not merged, joined, or denormalised. The customer receives the schema as it was at the point of extraction, with foreign key relationships preserved through matching column values.

### Encoding

- **UTF-8 without BOM.** Not UTF-8-BOM, not UTF-16, not Windows-1252, not Latin-1.
- The absence of a byte-order mark is deliberate. Modern tools handle UTF-8 correctly without one; older tools that require a BOM are increasingly rare and the customer can add one themselves if needed.

### Line endings

- **LF (`\n`)** between records.
- Not CRLF. Not CR alone.
- Cross-platform tools handle LF correctly; opinionated tools that insist on CRLF (some older Windows utilities) are minority cases.

### Header row

- **The first row of every file is a header row** containing column names.
- Column names match the source Postgres column names exactly, including case.
- Header row is never omitted, never made optional, never conditional.

## Field Delimiter and Quoting

Archiva follows **RFC 4180** with two deliberate refinements.

### Delimiter

- **Comma (`,`)** between fields.
- Not semicolon, not tab, not pipe. Regional variations are addressed by consuming tools, not by the export.

### Quoting

- Fields are **quoted with double quotes (`"`)** when they contain any of:
  - The field delimiter (`,`)
  - The line terminator (`\n`)
  - A double quote character (`"`)
- Fields that contain none of these characters are **not quoted**.
- **Producer preference:** unquoted where possible, for readability. Quoting only where required.

### Escaping quotes

- A double quote inside a quoted field is escaped by doubling it: `"He said ""yes""."`
- Backslash escaping is **not** used. This aligns with RFC 4180 and with spreadsheet conventions.

### Whitespace

- Leading and trailing whitespace **inside a field is preserved**. If the source has a trailing space, the export retains it.
- Leading and trailing whitespace **around a delimiter is not introduced**. `a,b,c` — never `a , b , c`.

## Data Type Representation

CSV has no native type system. Every value is a string on read. Archiva applies consistent conventions so consumers can parse types reliably.

### NULL

- Represented as **an empty field**, not the literal string `"NULL"`, `\N`, or an empty quoted string.
- Distinguishing an empty string from a NULL requires the schema documented in the manifest — CSV alone cannot represent the distinction.

### Booleans

- Represented as **`true` or `false`**, lowercase.
- Not `1`/`0`, not `Y`/`N`, not `TRUE`/`FALSE`.

### Integers

- Written as **decimal digits**, optionally prefixed by `-` for negative values.
- No thousand separators. No leading zeros.

### Decimals

- Written with a **`.` decimal separator**.
- No thousand separators.
- Precision preserved as it appears in Postgres — trailing zeros are retained if the column type specifies them.

### Dates

- **ISO 8601 date format: `YYYY-MM-DD`**.
- Not `DD/MM/YYYY`, not `MM/DD/YYYY`, not Postgres internal representations.

### Timestamps

- **ISO 8601 date-time format: `YYYY-MM-DDTHH:MM:SS`** for local timestamps.
- **ISO 8601 with timezone: `YYYY-MM-DDTHH:MM:SS+HH:MM`** or `Z` for UTC — used for `timestamp with time zone` Postgres columns.
- Microsecond precision preserved where present.

### JSON columns

Postgres `json` and `jsonb` columns are written as **a quoted, minified JSON string** in the CSV cell. Line breaks inside the JSON are not permitted. Consumers may parse the string as JSON.

### Binary data

- Postgres `bytea` columns are **base64-encoded** and written as a quoted string.
- Base64 without line breaks.

### Arrays

- Postgres array columns are written as **a quoted JSON array** — `"[\"a\", \"b\", \"c\"]"`.
- Not Postgres array syntax (`{a,b,c}`).

## File Size and Splitting

- Files are **not split by size**. Each table is one file, regardless of size.
- Very large tables (over 2 GB in CSV form) trigger a warning in the manifest but are not split. Customers with this constraint are directed to Parquet.
- Compression of the CSV files themselves is not applied at the file level. Compression is handled at the archive level (see below).

## Archive Packaging

- All CSV files for an engagement are packaged into a **single `.zip` archive** by default.
- `.tar.gz` is available on request.
- The archive root contains:
  - `manifest.json` — see [`manifest.md`](./manifest.md)
  - `csv/` — folder containing the `.csv` files
- Nested folders inside `csv/` are not used. Every `.csv` file sits at the top level of the `csv/` folder.

## Producer Behaviour

An Archiva Format producer generating CSV:

1. **Reads the schema** for each entitled table from Postgres
2. **Streams rows** in primary key order for reproducibility
3. **Writes** to a temporary file with the conventions above
4. **Computes SHA-256** of the completed file
5. **Records** the file in the manifest with row count, byte count, and checksum
6. **Includes** the file in the final `.zip` archive

Rows are never reordered by anything other than primary key. This is a reproducibility requirement — regenerating the same export from the same snapshot must produce a byte-identical file.

## Consumer Guidance

For customers or their engineers reading CSV exports:

- **Read as UTF-8.** Do not assume Windows-1252 or Latin-1.
- **Read the header row.** Do not assume column positions; consult column names.
- **Cross-reference the manifest.** The manifest declares data types for each column, since CSV itself does not carry types.
- **Verify the checksum** against the manifest before use.
- **Load into a typed system.** CSV is a transfer format, not a working format. Load into Postgres, SQLite, DuckDB, or a spreadsheet with declared types.

## Non-Goals

The following are deliberately not addressed by this specification:

- **Compression at the file level.** Handled at the archive level.
- **Multiple encodings.** UTF-8 is the only encoding produced.
- **Alternative delimiters.** Comma is the only delimiter produced.
- **Header customisation.** Column names are always the source names, unmodified.
- **Row filtering.** Format never applies its own filters; the customer receives what the engagement entitles them to.

## Rationale for Specific Choices

**Why UTF-8 without BOM:** the BOM causes more parsing failures than it prevents. Modern tools handle UTF-8 correctly without it.

**Why LF line endings:** cross-platform tools handle LF; opinionated Windows tools that require CRLF are a minority and easily addressed by the consumer.

**Why ISO 8601 dates:** unambiguous, machine-parseable, sortable as strings, internationally standardised. Every other date format has ambiguities.

**Why unquoted where possible:** readability. A record officer opening the CSV in a text editor should see clean columns, not fields drowning in unnecessary quotes.

**Why files are not split:** consistency and reproducibility. A file that splits at 500 MB one run and 501 MB the next is not reproducible. Customers with size constraints have Parquet.

## Version History

- **Draft (v1.0)** — Initial specification. Establishes conventions above.
