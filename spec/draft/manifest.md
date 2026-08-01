# Manifest Specification

**Version:** Draft (for inclusion in v1.0)

## Purpose

Every Archiva Format export includes a file called `manifest.json` at the root of the archive. The manifest is the single document that turns a folder of files into a documented, verifiable archive.

Without a manifest, an export is a collection of files. With one, it is a described, checksummed, versionable artefact — one that can be verified decades later without contacting Archiva.

The manifest carries what the file formats themselves cannot: type information for CSV columns, layout choices for JSON exports, cross-format inventory, and the checksums that prove nothing has been tampered with.

## Why JSON

The manifest is JSON regardless of which formats the export contains. JSON is the least demanding format to consume — no specialised tooling, no database dependency, no proprietary software. Any programming language, any operating system, and any text editor can read it.

This is a deliberate asymmetry: the customer's *data* can be in whichever format serves their onward use, but the *description of the data* is always in the format most universally readable.

## File Layout

- **Filename:** `manifest.json`
- **Location:** the root of the archive, not nested in any subfolder
- **Encoding:** UTF-8 without BOM
- **Line endings:** LF (`\n`)
- **Formatting:** pretty-printed with 2-space indent — the manifest is inspected by humans as often as by machines

## Structure

The manifest is a JSON object with the following top-level fields. All fields are required unless marked optional.

```json
{
  "format_version": "1.0",
  "generated_at": "2026-08-15T14:30:00Z",
  "engagement_reference": "eng_2026_08_lhac",
  "source_system": { ... },
  "formats_included": ["csv", "json"],
  "layouts": { ... },
  "tables": [ ... ],
  "reconciliation": { ... },
  "producer": { ... }
}
```

Each field is specified below.

## Field Reference

### `format_version` (required, string)

The version of the Format specification this manifest and its export were produced against.

- **Type:** string
- **Format:** semantic version — `"major.minor"` (e.g. `"1.0"`, `"1.2"`)
- **Example:** `"1.0"`

Consumers use this to determine which version of the specification governs the export they're reading. Every version's guarantees are honoured for as long as the customer holds the export.

### `generated_at` (required, string)

The date and time the export was generated, in UTC.

- **Type:** string
- **Format:** ISO 8601 with `Z` timezone suffix
- **Example:** `"2026-08-15T14:30:00Z"`

The generation time is not necessarily the same as the source data snapshot time. The manifest carries both where they differ — see `source_system.snapshot_at`.

### `engagement_reference` (required, string)

An opaque identifier tying the export to the source engagement. Not meaningful to third parties, but stable across regenerations of the same export.

- **Type:** string
- **Format:** lowercase alphanumeric with underscores; no whitespace
- **Example:** `"eng_2026_08_lhac"`

### `source_system` (required, object)

Describes the source system the data was extracted from.

```json
"source_system": {
  "type": "multivalue",
  "product": "synergysoft",
  "product_version": "12.4",
  "snapshot_at": "2026-08-14T23:00:00Z"
}
```

**Fields:**

- **`type`** (required, string) — the source system family. Valid values: `"multivalue"`, `"sql"`, `"api"`, `"file"`
- **`product`** (required, string) — the specific source product, lowercase. E.g. `"synergysoft"`, `"technologyone"`, `"xero"`
- **`product_version`** (optional, string) — the version of the source product at extraction time, if known
- **`snapshot_at`** (required, string) — ISO 8601 UTC timestamp of when the source data was snapshotted. May precede `generated_at` if the export was regenerated from a stored snapshot.

### `formats_included` (required, array of strings)

Which output formats are present in the archive. Each value corresponds to a subfolder in the archive.

- **Type:** array of strings
- **Valid values:** `"csv"`, `"json"`, `"parquet"`, `"sql"`
- **Example:** `["csv", "json"]`

An export must include at least one format. The archive contains a folder for each — `csv/`, `json/`, `parquet/`, or `sql/` — with the corresponding files inside.

### `layouts` (required, object)

Records format-specific layout choices made at generation time.

```json
"layouts": {
  "json": "flat"
}
```

**Current fields:**

- **`json`** (present only if JSON is included) — either `"flat"` or `"nested"`. See the [JSON specification](./json.md).

Layout choices for other formats will be added here if and when they become customer-configurable.

### `tables` (required, array of objects)

The core inventory. One entry per table exported, with per-format file details and column-level schema information.

```json
"tables": [
  {
    "name": "customers",
    "row_count": 47382,
    "columns": [
      {
        "name": "id",
        "type": "integer",
        "nullable": false,
        "primary_key": true
      },
      {
        "name": "name",
        "type": "text",
        "nullable": false
      },
      {
        "name": "created_at",
        "type": "timestamp",
        "nullable": false
      }
    ],
    "files": {
      "csv": {
        "path": "csv/customers.csv",
        "bytes": 3847264,
        "sha256": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855"
      },
      "json": {
        "path": "json/customers.json",
        "bytes": 5628493,
        "sha256": "b94d27b9934d3e08a52e52d7da7dabfac484efe37a5380ee9088f7ace2efcde9"
      }
    }
  }
]
```

**Per-table fields:**

- **`name`** (required, string) — the table name, lowercase, matching the source
- **`row_count`** (required, integer) — the number of rows in the table
- **`columns`** (required, array of objects) — see below
- **`files`** (required, object) — one entry per format included, keyed by format name

**Per-column fields:**

- **`name`** (required, string) — column name, matching the source exactly
- **`type`** (required, string) — Archiva's normalised type. Valid values: `"boolean"`, `"integer"`, `"bigint"`, `"decimal"`, `"text"`, `"date"`, `"timestamp"`, `"timestamp_tz"`, `"json"`, `"array"`, `"bytea"`
- **`nullable`** (required, boolean) — whether the column allows NULL
- **`primary_key`** (optional, boolean) — `true` if this column is part of the primary key; omitted otherwise
- **`decimal_precision`** (required if `type` is `"decimal"`, integer) — total digits
- **`decimal_scale`** (required if `type` is `"decimal"`, integer) — digits after the decimal point
- **`array_element_type`** (required if `type` is `"array"`, string) — the element type using the same type vocabulary

**Per-file fields (inside `files`):**

- **`path`** (required, string) — the file path relative to the archive root
- **`bytes`** (required, integer) — file size in bytes
- **`sha256`** (required, string) — hex-encoded SHA-256 checksum of the file contents

### `reconciliation` (required, object)

Describes the reconciliation performed between the source system and the exported data. Every export is expected to have been reconciled before delivery — the manifest carries the summary.

```json
"reconciliation": {
  "performed_at": "2026-08-15T13:45:00Z",
  "method": "row_count_and_control_totals",
  "outcome": "matched",
  "report_reference": "recon_2026_08_lhac_r1"
}
```

**Fields:**

- **`performed_at`** (required, string) — ISO 8601 UTC timestamp of the reconciliation
- **`method`** (required, string) — the reconciliation approach used. Valid values: `"row_count"`, `"row_count_and_control_totals"`, `"row_count_and_sample_verification"`, `"full_field_comparison"`
- **`outcome`** (required, string) — `"matched"`, `"matched_with_variances"`, or `"discrepancy"`
- **`report_reference`** (required, string) — opaque identifier for the reconciliation report, held by Archiva

If the outcome is anything other than `"matched"`, the export includes an additional file `reconciliation_notes.md` at the archive root documenting the variances or discrepancies.

### `producer` (required, object)

Identifies the software that produced the export.

```json
"producer": {
  "name": "Archiva Format Producer",
  "version": "1.0.0",
  "produced_by": "Archiva Group Limited"
}
```

**Fields:**

- **`name`** (required, string) — the producer software name
- **`version`** (required, string) — the producer software version
- **`produced_by`** (required, string) — the entity operating the producer

## Complete Example

```json
{
  "format_version": "1.0",
  "generated_at": "2026-08-15T14:30:00Z",
  "engagement_reference": "eng_2026_08_lhac",
  "source_system": {
    "type": "multivalue",
    "product": "synergysoft",
    "product_version": "12.4",
    "snapshot_at": "2026-08-14T23:00:00Z"
  },
  "formats_included": ["csv", "json"],
  "layouts": {
    "json": "flat"
  },
  "tables": [
    {
      "name": "customers",
      "row_count": 47382,
      "columns": [
        { "name": "id", "type": "integer", "nullable": false, "primary_key": true },
        { "name": "name", "type": "text", "nullable": false },
        { "name": "balance", "type": "decimal", "decimal_precision": 12, "decimal_scale": 2, "nullable": false },
        { "name": "created_at", "type": "timestamp_tz", "nullable": false }
      ],
      "files": {
        "csv": {
          "path": "csv/customers.csv",
          "bytes": 3847264,
          "sha256": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855"
        },
        "json": {
          "path": "json/customers.json",
          "bytes": 5628493,
          "sha256": "b94d27b9934d3e08a52e52d7da7dabfac484efe37a5380ee9088f7ace2efcde9"
        }
      }
    }
  ],
  "reconciliation": {
    "performed_at": "2026-08-15T13:45:00Z",
    "method": "row_count_and_control_totals",
    "outcome": "matched",
    "report_reference": "recon_2026_08_lhac_r1"
  },
  "producer": {
    "name": "Archiva Format Producer",
    "version": "1.0.0",
    "produced_by": "Archiva Group Limited"
  }
}
```

## Consumer Guidance

For customers or third parties reading a Format export:

- **Read the manifest first.** It tells you what's in the archive and how to interpret each file.
- **Check `format_version`.** The specification for that version governs everything else.
- **Verify checksums.** For every file in the export, compute the SHA-256 and compare to the value in the manifest. Any mismatch means the file has been altered since generation.
- **Consult column types.** CSV files don't carry types — the manifest is where they live. JSON files carry native types but decimal columns are strings for precision — the manifest tells you which.
- **Check the reconciliation outcome.** If it's anything other than `"matched"`, read `reconciliation_notes.md` before relying on the data.

## Producer Behaviour

An Archiva Format producer writing the manifest:

1. **Assembles** all required fields from the engagement configuration and export state
2. **Computes** the SHA-256 of every produced file
3. **Sorts** tables alphabetically by name for reproducibility
4. **Sorts** columns within each table by their source ordinal position
5. **Writes** `manifest.json` to the archive root, pretty-printed with 2-space indent, UTF-8 without BOM
6. **Includes** the manifest as the last file added to the archive, after all data files are finalised

Reproducibility requirement: given the same source snapshot and engagement configuration, two independent runs must produce byte-identical manifests. This means no timestamps other than `generated_at` and `performed_at` (which are recorded), and no unstable ordering anywhere in the structure.

## Non-Goals for v1.0

The following are deliberately not included in v1.0 of the manifest:

- **Cryptographic signing.** Manifests are checksummed at the file level but not signed. A published JSON Schema and manifest signing are planned for v1.1 and v2.0 respectively.
- **JSON Schema publication.** The manifest structure is documented here but not published as a machine-readable JSON Schema document. Planned for v1.1.
- **Change history within the manifest.** Regenerated exports get new manifests; the manifest does not carry history of prior generations. The engagement reference is stable across regenerations if the customer needs to correlate.
- **Producer-specific extensions.** The manifest structure is fixed. Producers do not add custom fields. Extensions to the specification go through the normal contribution process.

## Rationale for Specific Choices

**Why JSON, not YAML or TOML:** JSON has the widest tooling support. Every language, every editor, every operating system reads it natively. YAML is more human-readable but has parsing pitfalls (implicit type coercion, whitespace sensitivity). TOML is elegant but less universally supported.

**Why per-file checksums, not per-archive:** individual file checksums let a consumer verify each file independently. A single archive-level checksum only tells you the archive is intact, not which file was altered if the check fails.

**Why SHA-256, not MD5 or SHA-1:** SHA-256 is the current professional standard. MD5 and SHA-1 are considered cryptographically broken. SHA-512 offers no meaningful advantage for integrity checking and produces longer strings.

**Why the source system is a structured object:** the source system is a first-class piece of context. Consumers frequently ask *"where did this data originally come from?"* — the answer belongs in the manifest, structured, not buried in an engagement reference.

**Why reconciliation is required:** every Archiva export must have been reconciled before delivery. The manifest recording this is what makes the completeness guarantee verifiable. An export without reconciliation is not a valid Archiva export.

**Why sorted alphabetically:** reproducibility. Same input, same output, byte-identical. Any unstable ordering breaks this.

## Version History

- **Draft (v1.0)** — Initial specification. Establishes structure above.
