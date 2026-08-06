# Roadmap

The path from the current draft to v1.0 and beyond.

## Where we are

Six specification documents are complete in draft in [`spec/draft/`](./spec/draft/):

- CSV — universal readable format
- JSON — structured export with layout choice
- Manifest — the `manifest.json` structure
- SQL — Postgres dialect database dump
- Parquet — columnar analytical format
- Guarantees — the six commitments Archiva makes

All specifications are considered stable enough to build against, but formal v1.0 release awaits validation.

## Path to v1.0

Three things must be true before the specifications leave draft:

1. **The Format producer exists and runs.** No output has been produced against these specifications yet. Real production output will surface details the specifications should address.
2. **At least one real export has been produced.** Reviewing an actual export against the specifications is the final test that the specifications work in practice.
3. **The specifications have been re-reviewed.** After producer implementation, every specification gets a fresh read to catch anything that turned out to be impractical or unclear.

When these three are complete, `spec/draft/` becomes `spec/v1.0/`, a v1.0 GitHub release is tagged, and every subsequent export is bound by v1.0's guarantees.

## After v1.0

### v1.1 — planned

Additive changes. Backwards compatible. Existing v1.0 exports remain valid.

- **Publish JSON Schema for the manifest.** The manifest structure is fully specified in `manifest.md` but is not yet available as a machine-readable JSON Schema document. v1.1 publishes one, enabling automated manifest validation.
- **Customer-selectable SQL dialect.** v1.0 produces Postgres SQL only. v1.1 adds MySQL, SQL Server, and SQLite as customer-selectable dialects at export time.
- **Postgres `COPY FROM` fast-load variant for SQL exports.** Optional variant using Postgres-native fast load format, dramatically faster to load but Postgres-only.
- **Configurable batch size for SQL INSERTs.** v1.0 fixes multi-row INSERTs at 1000 rows per statement. v1.1 makes this configurable per engagement.

### v2.0 — planned

Breaking changes. Justified by structural improvement that cannot be delivered additively.

- **Cryptographic signing of manifests.** File-level checksums prevent tampering with data files. v2.0 adds signature-based tampering detection for the manifest itself, closing the last integrity gap. Requires public key infrastructure for signature verification, which is why it belongs at a major version boundary.

## What is not planned

Some things are deliberately absent from this roadmap:

- **Real-time or continuous exports.** Ongoing sync between source and archive is Continuum's territory, not Format's.
- **Immutability enforcement beyond checksums.** Write-once storage or blockchain-anchored proofs of receipt are not part of Format's scope. Customers who require them can layer them on top.
- **Guarantees against source system errors.** Format faithfully reproduces what the source held. Source correction is a separate concern.
- **Long-term producer availability guarantees.** The specifications are stable across versions. The software producing exports may evolve.

## How to influence the roadmap

Substantial changes to the specification are discussed as issues before drafting. See [`CONTRIBUTING.md`](./CONTRIBUTING.md).

The roadmap reflects Archiva's current planning. If you have specific needs that would shape it, we want to hear about them.
