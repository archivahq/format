# JSON Format Specification

**Version:** Draft (for inclusion in v1.0)

## Purpose

JSON is the structured export format. Where CSV serves the widest possible audience of tools and readers, JSON serves the *structured* audience — applications, APIs, and integrations that need machine-readable data with a clear shape. Every Archiva engagement offers JSON as an alternative or complement to CSV when the customer's onward use is application-driven.

## What JSON is best for

- Feeding into applications that consume a customer's records programmatically
- Integrations with modern web platforms and APIs
- Data with genuine hierarchical relationships (a customer with their invoices with their line items)
- Systems that read structured input natively and would treat CSV as a step backwards

## What JSON is worst for

- Very large flat datasets — Parquet is more efficient
- Human review at the row level — CSV is easier to open and skim
- Direct import into a spreadsheet — CSV is what spreadsheets expect
- Long-term storage where file size matters — JSON is verbose

Customers with these needs are directed to CSV or Parquet.

## Layout — the one customer choice

JSON is the only format where the customer chooses the *structure* of the export at generation time. Two layouts are supported and both are first-class citizens of this specification.

### Flat layout (default)

One JSON file per source table. Each file contains an array of objects, one object per row. Foreign key relationships are preserved as values but not expanded — the consumer joins tables themselves if they need to.

```json
[
  { "id": 42, "name": "Jane Cooper", "created_at": "2024-08-15" },
  { "id": 43, "name": "John Smith", "created_at": "2024-09-01" }
]
```

Best for: engineers loading data into their own system, database rebuilds, structured archives.

### Nested layout

Relationships are expanded inline. Related records are included as nested objects or arrays inside their parent, based on foreign key relationships declared in the manifest.

```json
[
  {
    "id": 42,
    "name": "Jane Cooper",
    "invoices": [
      {
        "id": 1001,
        "amount": "1250.00",
        "lines": [
          { "id": 5001, "description": "Consulting", "amount": "1000.00" },
          { "id": 5002, "description": "Materials", "amount": "250.00" }
        ]
      }
    ]
  }
]
```

Best for: application feeds, document-store loading, systems that consume whole customer records at once.

### How the customer chooses

The layout is specified at export generation time, not per-file. An export is entirely flat or entirely nested; the two are not mixed. If a customer wants both, two separate exports are produced.

The chosen layout is recorded in the manifest so consumers know what shape to expect.

### Which layout is default

If the customer expresses no preference, **flat layout** is produced. Reasons:

- Flat is more predictable and easier to consume for engineering audiences
- Nested requires Archiva to make interpretive decisions about which relationships to expand
- Flat matches the CSV layout, which most engagements will produce alongside JSON

Customers wanting nested layout indicate this at engagement scoping.

## File Layout

### One file per source table

Whether flat or nested, each Postgres table (or root table, in the nested case) produces one `.json` file. Files are named after the table, in lowercase:
customers.json
invoices.json
invoice_lines.json

For nested layouts, only root tables have their own file — child records live inside their parents. Which tables are treated as roots is declared in the manifest.

### Encoding

- **UTF-8 without BOM.** Same rationale as CSV — modern default, no legacy compatibility artefacts.

### Line endings

- **LF (`\n`)** between lines. JSON parsers don't care, but consistency across formats matters.

### Formatting

- **Pretty-printed with 2-space indent.**
- One key per line for objects and arrays with more than one element.
- Human-readable when opened directly.

Rationale: engineering audiences frequently inspect JSON exports visually. The file size overhead of pretty-printing is negligible after zip compression, and readability is genuinely valuable.

## Object Structure

### Wrapper shape

The top level of every JSON file is an **array of objects**, not an object with metadata wrapping records.

```json
[
  { "id": 1, ... },
  { "id": 2, ... }
]
```

Metadata about the file (row count, source table, generation time) lives in the manifest, not in the file itself. This keeps the file shape predictable and simple to consume.

### Object keys

- Keys match the source Postgres column names exactly, including case
- No renaming, no camelCase conversion, no prefixing
- The manifest declares column types for consumers who need to interpret string values

### Row ordering

- Rows are written in **primary key order** for reproducibility
- Same source snapshot produces byte-identical output — critical for the integrity guarantee

## Data Type Representation

JSON has richer native types than CSV, but is not without pitfalls. Archiva applies consistent conventions.

### NULL

- Native JSON `null`.
- Not the string `"null"`, not omission of the key.
- Every column present in the schema appears in every row, with `null` where the value is missing.

### Booleans

- Native JSON `true` / `false`, lowercase.

### Integers

- Native JSON numbers, no quotes.
- JSON integers are handled safely by parsers up to 2^53 — Archiva's identifier columns comfortably fit within this range.
- Integer columns whose values exceed 2^53 (rare, and explicitly flagged in the manifest) are written as strings.

### Decimals

- **Written as strings, not JSON numbers.**
- Example: `"amount": "1250.00"`, not `"amount": 1250.00`.

Rationale: JSON's numeric type has no fixed precision guarantee. A decimal value like `0.1 + 0.2` in JavaScript produces `0.30000000000000004`. For financial and quantity data, this is unacceptable. Writing decimals as strings preserves exact precision. Consumers parse the string to their own precise decimal type.

The manifest declares which columns are numeric-as-string so consumers know to parse them.

### Dates

- ISO 8601 date strings: `"2024-08-15"`.
- Same convention as CSV.

### Timestamps

- ISO 8601 datetime strings.
- Local timestamps: `"2024-08-15T14:30:00"`.
- With timezone: `"2024-08-15T14:30:00+10:00"` or `"2024-08-15T14:30:00Z"`.

### JSON columns

- Postgres `json` and `jsonb` columns are **preserved as native JSON** — the value is emitted inline, not stringified.
- If the source has `{"foo": "bar"}` as a JSONB value, the export has `"config": {"foo": "bar"}` — not `"config": "{\"foo\": \"bar\"}"`.

### Arrays

- Postgres array columns are written as **native JSON arrays**.
- `["a", "b", "c"]`, not the Postgres syntax or a stringified representation.

### Binary data

- `bytea` columns are **base64-encoded** and written as a string.
- Base64 without line breaks.
- Same convention as CSV.

## Archive Packaging

- All JSON files for an engagement are packaged into a **single `.zip` archive** by default.
- `.tar.gz` on request.
- The archive root contains:
  - `manifest.json` — see [`manifest.md`](./manifest.md)
  - `json/` — folder containing the `.json` files
- Nested folders inside `json/` are not used.

## Producer Behaviour

An Archiva Format producer generating JSON:

1. **Reads the layout choice** from the engagement configuration (flat or nested)
2. **Reads the schema** for each table from Postgres
3. **Streams rows** in primary key order
4. **Applies type conventions** — decimals to strings, dates to ISO 8601, arrays and JSON preserved natively
5. **Writes** to a temporary file, pretty-printed
6. **Computes SHA-256** of the completed file
7. **Records** the file in the manifest with row count, byte count, checksum, and layout used
8. **Includes** the file in the final `.zip` archive

Layout choice is recorded in the manifest so consumers know what to expect before parsing.

## Consumer Guidance

For customers or their engineers reading JSON exports:

- **Read as UTF-8.** Do not assume any other encoding.
- **Check the manifest for the layout used.** Flat and nested exports have different parsing requirements.
- **Parse decimal columns explicitly.** They are strings, not JSON numbers, for precision reasons. Use a decimal-aware parser.
- **Verify the checksum** against the manifest before use.
- **Load into a typed system** if the data will be queried or analysed. JSON is a transfer format.

## Non-Goals

The following are deliberately not addressed by this specification:

- **JSON Lines / newline-delimited JSON.** Requested as a variant occasionally; not produced by default. Customers with streaming pipelines can transform the standard JSON output.
- **Schema Draft / JSON Schema files.** The manifest carries schema information in a simpler form. JSON Schema output can be requested as a supplementary artefact.
- **Minified output.** Pretty-printed is the default. Minified is not offered — file size savings are marginal after zip compression, and the readability loss is real.
- **Custom key naming.** Column names are preserved exactly as they are in the source.

## Rationale for Specific Choices

**Why flat is the default:** predictable, simpler, easier to consume without interpretive decisions. Matches CSV structure so engagements offering both feel consistent.

**Why decimals as strings:** JSON's number type cannot guarantee precision. For financial data this is a non-negotiable correctness issue. Consumers accepting the small parse-time cost gain exact values.

**Why pretty-printed:** JSON is often inspected by humans. Zip compression eliminates the file size overhead. Readability is worth preserving.

**Why one file per table:** consistency with CSV. Streaming-friendly for large tables. Simple mental model for consumers.

**Why native JSON for JSON columns:** stringifying a JSON value inside a JSON file is technically valid but wasteful and confusing. Preserving the value natively is what a well-formed JSON export should do.

**Why lowercase filenames:** cross-platform consistency. Case-sensitive filesystems (Linux) and case-insensitive filesystems (macOS default, Windows) both handle lowercase reliably.

## Version History

- **Draft (v1.0)** — Initial specification. Establishes conventions above.
