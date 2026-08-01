# Format Specification — Draft

This is the current in-development version of the Archiva Format specification. Once stable, it will be released as a numbered version (`v1.0/`) and this draft folder will begin the next revision.

**Status:** Draft. Nothing in this folder is a released commitment yet.

## What lives in this folder

When complete, this specification version will include:

- `README.md` — this file: overview, rationale, and what has changed since the last version
- `manifest.md` — the manifest structure and required fields for every Format export
- `csv.md` — the CSV format specification: encoding, quoting, headers, dates, edge cases
- `parquet.md` — the Parquet format specification: schema encoding, compression, versioning
- `json.md` — the JSON format specification: flat and nested variants, structure
- `sql.md` — the SQL format specification: DDL, INSERT statements, dialect handling
- `guarantees.md` — the quality commitments Archiva makes for exports at this version

## Current status of each section

- `README.md` — in progress *(you're reading it)*
- `manifest.md` — planned
- `csv.md` — planned
- `parquet.md` — planned
- `json.md` — planned
- `sql.md` — planned
- `guarantees.md` — planned

## Rationale for v1.0

The first released version of the Format specification exists to fix four things:

1. **Four output formats** — CSV, Parquet, JSON, and SQL. Chosen to cover the majority of customer needs without proliferating.
2. **A manifest structure** — every export includes `manifest.json` at the archive root, sufficient to verify the export decades later without contacting Archiva.
3. **Producer behaviour** — how each format is generated: encoding, quoting conventions, compression, schema handling, dialect choices.
4. **Quality guarantees** — completeness, integrity, portability, reproducibility, documentation. What Archiva commits to for every export.

Later versions may add formats, expand the manifest, or refine producer behaviour. This first version is deliberately narrow.

## Contributing to the draft

See the repository-level [`CONTRIBUTING.md`](../../CONTRIBUTING.md). Substantial changes to draft sections are discussed as issues before drafting.
