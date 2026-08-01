# Format

The public specification for **Archiva Format** — the file export product.

Archiva Format is one of three ways an Archiva engagement delivers data from the pipeline to the customer. Where **Vault** holds the record long-term and **Continuum** syncs it live to a target system, **Format** returns the data as files — CSV, Parquet, JSON, and SQL — in the shape the customer needs, in formats they own.

This repository is the *public specification* of Format. It documents the file layouts, the manifest structure, and the quality guarantees Archiva makes for every export. Any customer, third-party engineer, or successor provider should be able to open a Format export and understand it fully from what's documented here — without contacting Archiva.

The implementation itself is not published; only the specification is.

## What this repository contains

- **[`spec/`](./spec)** — the specification, versioned. Each folder is a released version (`v1.0/`, `v1.1/`, etc.), with the current draft in `spec/draft/`
- **[`examples/`](./examples)** — small, illustrative sample exports demonstrating each format
- **[`CHANGELOG.md`](./CHANGELOG.md)** — the version history of the specification itself

## What this repository is not

- Not the source code that produces Format exports — that stays private
- Not documentation for how to use Archiva as a product — that lives at [archivahq.com](https://archivahq.com)
- Not a general reference for CSV / Parquet / JSON / SQL — the standards for those live at their respective specifications

## The four formats, briefly

The full detail is in [`spec/`](./spec), but for a quick orientation:

- **CSV** — universal readability. The lowest common denominator. Anything you'd open in Excel.
- **Parquet** — columnar binary format. For analytics teams and modern data warehouses.
- **JSON** — structured, hierarchical text. For applications, APIs, and integrations.
- **SQL** — a database dump. DDL plus INSERT statements. For rebuilding a queryable database.

Each is described plainly in the specification, including what it's best for, what its weaknesses are, and how Archiva produces it.

## The manifest

Every Format export includes a `manifest.json` at the archive root. This is what turns a folder of files into a documented, verifiable archive — table inventory, checksums, format version, schema summary, and a producer signature attesting the export was produced by Archiva.

A customer can verify a Format export decades later without contacting Archiva. That's the point.

The manifest structure is fully specified in [`spec/`](./spec). It is the single most important artefact in any Format export.

## Guarantees

Archiva makes the following guarantees for every Format export produced against this specification:

- **Completeness** — the export contains everything the reconciliation report accounts for
- **Integrity** — every file has a SHA-256 checksum in the manifest; tampering is detectable
- **Portability** — the formats are open specifications; no proprietary tools required
- **Reproducibility** — a Format export can be regenerated from the same source snapshot and will be byte-identical
- **Documentation** — the manifest is sufficient to reconstruct meaning without contacting Archiva

## Versioning

The specification is versioned independently of the Archiva platform.

- **Major** (`v1.0` → `v2.0`) — breaking changes to file layout, manifest structure, or checksum method. Rare.
- **Minor** (`v1.0` → `v1.1`) — additive changes. New optional manifest fields, new format variants, new supported target languages. Backwards compatible.

Every export declares which version it was produced against. Consumers can rely on that version's guarantees for as long as they hold the export.

## Reading the specification

Start with [`spec/draft/README.md`](./spec/draft/README.md) for the current in-development version, or pick a released version folder for a locked, stable specification.

Each version contains:

- `README.md` — the version's overview and rationale
- `manifest.md` — the manifest structure and required fields
- `csv.md`, `parquet.md`, `json.md`, `sql.md` — one file per format, describing its layout and producer behaviour
- `guarantees.md` — the quality commitments Archiva makes for that version

## Contributing

Corrections, clarifications, and improvements to the specification are welcome. See [`CONTRIBUTING.md`](./CONTRIBUTING.md).

Additions to the specification (new formats, new manifest fields, new delivery methods) are considered but not accepted lightly — the specification is a commitment to every past and future customer, and expanding it carries obligations. Substantial changes are discussed as issues first.

## About Archiva

[Archiva Group](https://archivahq.com) helps public sector organisations extract and archive data from legacy ERP systems into open formats they own — for audit and record-keeping, or for onward migration.

Registered in the United Kingdom. Operating in Australia, New Zealand and the United Kingdom.

## Licence

The specification text and documentation are licensed under [Creative Commons Attribution 4.0 International](./LICENSE). Sample code and example manifest structures, where present, are licensed under [MIT](./LICENSE-CODE).
