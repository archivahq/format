# Changelog

All notable changes to the Archiva Format specification are documented here.

The format follows [Semantic Versioning](https://semver.org/): major version changes are breaking, minor versions are backwards-compatible additions, patches are corrections that do not affect behaviour.

## Unreleased

Current in-development version. Working documents live in [`spec/draft/`](./spec/draft/).

Six specification documents are complete in draft:

- **CSV** — universal readable format specification
- **JSON** — structured export with customer-selectable flat or nested layout
- **Manifest** — the `manifest.json` structure that describes every export
- **SQL** — Postgres dialect, `schema.sql` + `data/*.sql` structure with deferrable foreign keys
- **Parquet** — Zstandard compression, 128 MB row groups, column statistics
- **Guarantees** — six commitments with enforcement, verification, and remediation for each

Draft status will be maintained until:

1. The Format producer is built and running
2. At least one real export has been produced against these specifications
3. Specifications have been reviewed against real production behaviour

At that point v1.0 will be formally released. See [`ROADMAP.md`](./ROADMAP.md) for the path forward.

## Change history

### 2026-08-04

- Six specification documents complete in draft
- Jurisdiction added as sixth guarantee
- Parquet compression locked to Zstandard
- Parquet row group size locked to 128 MB
- Parquet column statistics enabled by default
- SQL structure locked: `schema.sql` for DDL plus `data/*.sql` per table for INSERTs
- SQL foreign keys marked as `DEFERRABLE INITIALLY DEFERRED` for load-order independence
- SQL INSERTs standardised as multi-row at 1000 rows per statement
- SQL transactions scoped per-table for clear failure boundaries

### 2026-07-31

- Initial repository structure established
- README, CONTRIBUTING, LICENSE (CC BY 4.0), LICENSE-CODE (MIT) committed
- `spec/draft/` directory structure established
- CSV, JSON, and manifest specifications drafted
- Four output formats confirmed: CSV, JSON, SQL, Parquet
