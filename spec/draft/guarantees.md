# Guarantees

**Version:** Draft (for inclusion in v1.0)

## Purpose

Every Archiva Format export is produced against six commitments. This document specifies what each commitment means in technical terms, how it is enforced during production, how a consumer can verify it independently, and how a violation would be detected and remediated.

The guarantees are the reason Format exists as a *product* rather than a *convenience*. Without them, an export is a folder of files. With them, an export is a documented, verifiable, and portable artefact — one that will still be trustworthy in ten years, whether or not Archiva still exists.

## The six guarantees

1. **Completeness** — the export contains everything the reconciliation report accounts for
2. **Integrity** — every file has a SHA-256 checksum in the manifest; tampering is detectable
3. **Portability** — the formats are open specifications; no proprietary tools required
4. **Reproducibility** — the same source snapshot produces byte-identical output
5. **Documentation** — the manifest is sufficient to reconstruct meaning without contacting Archiva
6. **Jurisdiction** — data does not cross a national boundary without explicit written instruction

Each is specified in full below.

---

## 1. Completeness

### What it means

An Archiva Format export contains every record accounted for in the reconciliation report performed at extraction. If reconciliation showed 47,382 customer records in the source system at snapshot time, the export contains 47,382 customer records — no more, no fewer.

Completeness is a statement about *what was produced*, verified against *what the source held at the moment of extraction*. It does not extend to records created in the source system after the snapshot.

### How it is enforced

- The reconciliation report is generated before the Format producer runs
- The producer reads directly from the reconciled Postgres snapshot, not from any intermediate cache
- Every row extracted is counted per table; final counts are recorded in the manifest
- The producer will not finalise an export where its table counts disagree with the reconciliation report
- The `reconciliation` field in the manifest carries the reconciliation method, outcome, and report reference

### How a consumer verifies it

- Read the manifest's `reconciliation` field to confirm outcome is `matched`
- Read each table's `row_count` in the manifest
- Load the export into any queryable system and count rows per table
- Confirm the loaded row counts match the manifest

### How a violation would be handled

If the reconciliation outcome is `matched_with_variances` or `discrepancy`, the export includes `reconciliation_notes.md` at the archive root documenting every variance. An export with unresolved discrepancies is delivered *with the discrepancies flagged*, not withheld — the customer decides whether to accept, reject, or request re-extraction.

An export marked `matched` where subsequent verification reveals a row-count mismatch is a serious defect. Archiva would investigate the pipeline, produce a corrected export, and note the incident in the engagement's audit record. The prior export would be superseded by an explicit reference in the new manifest.

---

## 2. Integrity

### What it means

Every file in an Archiva Format export has a cryptographic checksum recorded in the manifest. If any file is altered after generation — deliberately or accidentally — the alteration is detectable by anyone reading the export.

Integrity is a statement about *the export as delivered*. It does not prevent tampering with the manifest itself; that is addressed in *jurisdiction* and, in future versions, in cryptographic signing.

### How it is enforced

- Every file produced (data files, `manifest.json` itself is written last) is hashed using SHA-256
- Hashes are computed on the finalised file contents, not on any intermediate buffer
- Hashes are written into the manifest's per-file `sha256` field
- The manifest is written last, once all data files exist and their checksums are known
- The complete archive is produced from the finalised files without further modification

### How a consumer verifies it

For every file listed in the manifest's `tables[].files` entries:

- Read the file from the archive
- Compute its SHA-256 checksum using any standard cryptographic library
- Compare to the value in the manifest
- Any mismatch indicates the file has been altered since generation

For the manifest itself (v1.0): no cryptographic verification is possible without also having a trusted reference. This is addressed in *v1.1 (JSON Schema publication)* and *v2.0 (manifest signing)*.

### How a violation would be handled

A checksum mismatch on receipt indicates the file has been altered in transit or at rest. The customer should not treat the affected file as authoritative. Archiva would investigate the delivery chain, produce a fresh export (which will have new checksums), and — in serious cases — investigate whether the delivery method itself is trustworthy.

---

## 3. Portability

### What it means

An Archiva Format export can be read by any competent technical team using only open specifications and standard tooling. No proprietary Archiva software is required to open, verify, or work with the export at any point after delivery.

Portability is what makes Format a genuinely customer-owned artefact rather than a license to view the customer's own data.

### How it is enforced

- All formats produced (CSV, Parquet, JSON, SQL) are governed by public specifications maintained by open standards bodies or open communities
- No proprietary compression, encoding, or transformation is used at any stage
- The manifest is JSON — the least demanding format to consume, with universal tooling support
- File and folder naming conventions are documented in this specification and use only characters portable across every major operating system

### How a consumer verifies it

- Open any file in the export using a tool of your choice — a text editor for CSV / JSON / SQL, a Parquet-capable analytical tool for Parquet
- Confirm the file opens without proprietary software
- Confirm the manifest is valid JSON parseable by any standard JSON library
- Load the data into any downstream system without transformation beyond format-specific parsing

### How a violation would be handled

A Format export that requires proprietary tooling to read would be a defect against this specification. Archiva would investigate, document, and remediate — likely by producing a corrected export. In the unlikely event that a format-standard change made a previously-portable export effectively unreadable, Archiva would publish guidance on continued access and would produce a re-exported version on request.

Portability is asserted at the version of the specification the export was produced against. Format v1.0 exports remain portable under v1.0 rules for as long as the customer holds them, regardless of what later specification versions do.

---

## 4. Reproducibility

### What it means

Given the same source snapshot and the same producer configuration, two independent runs of the Format producer will produce byte-identical exports. The archive checksum will be the same. Every file checksum will be the same. The manifest will be the same, except for the `generated_at` timestamp and any timestamps recording the runs.

Reproducibility is what makes it possible for the customer, an auditor, or a successor provider to *verify* that an export they hold is what Archiva claims it is.

### How it is enforced

- Rows are extracted in primary key order — never in insertion order, never in physical-storage order
- Tables in the manifest are sorted alphabetically by name
- Columns within each table are ordered by source ordinal position
- File hashing uses a deterministic algorithm (SHA-256) with no randomised input
- JSON output is pretty-printed with a fixed 2-space indent, canonical key ordering
- No timestamps other than the explicitly declared `generated_at` and `reconciliation.performed_at` are written into any file

### How a consumer verifies it

If the consumer holds two supposed regenerations of the same export:

- Compare the archive checksums
- Compare each per-file checksum against the manifest
- Verify that any differences between the manifests are confined to the declared timestamp fields

Byte-for-byte identity across the two runs (excluding declared timestamps) confirms reproducibility.

### How a violation would be handled

Non-reproducible output — where two runs of the same source snapshot produce different exports — is a serious defect against this specification. It undermines every other guarantee, because it makes independent verification impossible. Archiva would investigate the producer implementation, correct the source of nondeterminism, and re-verify against known-good test cases before returning to production.

---

## 5. Documentation

### What it means

The manifest of an Archiva Format export contains sufficient information for a competent technical team to reconstruct the meaning of the exported data without contacting Archiva.

This includes: which tables are present, what columns each table contains, what types those columns are, which columns are nullable, which are primary keys, what the reconciliation position was at extraction, which source system produced the data, and when the extraction occurred.

Documentation is what makes an Archiva Format export usable *decades* after generation, when institutional knowledge about the engagement has been lost.

### How it is enforced

- Every column in every table is documented in the manifest with its normalised type, nullability, and (where applicable) primary key membership
- Decimal columns include precision and scale
- Array columns include element type
- Timestamp columns declare whether they carry timezone context
- The source system is identified by family, product, and (where known) version
- The reconciliation reference points to a report Archiva retains, allowing successor providers to request its content

### How a consumer verifies it

- Open the manifest
- Confirm every table has a `columns` array with type, nullability, and (where applicable) primary key and decimal precision annotations
- Confirm the `source_system` field carries type, product, and snapshot timestamp
- Confirm the `reconciliation` field carries method, outcome, and reference

A consumer who can build a working understanding of the data from the manifest alone has verified the documentation guarantee.

### How a violation would be handled

Missing or incorrect manifest documentation is a defect against this specification. Archiva would produce a corrected manifest, verify it against the export contents, and — where the deficiency was structural rather than local — re-review other recent exports for the same issue.

---

## 6. Jurisdiction

### What it means

Data in an Archiva Format export does not cross a national boundary without explicit written instruction from the customer. If the source data was extracted from a system in Australia, the export is produced in Australia, held in Australia, and delivered to the customer through infrastructure that does not route the data outside Australia.

The same principle applies to New Zealand and the United Kingdom, each treated independently.

Jurisdiction is what makes Archiva's sovereignty claims verifiable rather than aspirational. It is not a policy commitment; it is a technical property of how exports are produced and delivered.

### How it is enforced

- Production infrastructure for each jurisdiction sits in cloud regions within that jurisdiction
- Australian engagements produce exports in Australia East region
- New Zealand engagements produce exports in New Zealand North region
- United Kingdom engagements produce exports in UK South region
- Delivery URLs (signed download links, storage bucket references) resolve to regional endpoints
- Backup and disaster recovery for regional infrastructure remain within the same jurisdiction
- Movement of an export across a jurisdiction requires an explicit written request from the customer, recorded in the engagement audit trail

### How a consumer verifies it

- Read the manifest's `producer` field — the `produced_by` entry declares Archiva Group Limited as the operating entity
- Confirm the delivery endpoint (download URL, bucket region) resolves within the expected jurisdiction — typically via a network trace or a DNS lookup against the delivery URL
- Request written confirmation of the production region from Archiva; this is available on request for any engagement

For customers on the *"in your environment"* deployment model, jurisdiction is enforced trivially: the export is produced inside infrastructure the customer owns and controls, and never leaves it.

### How a violation would be handled

Cross-jurisdiction data movement without written customer instruction would be a serious breach of this specification and, depending on the jurisdiction, potentially a legal matter. Archiva would investigate the incident, notify the customer immediately, notify the applicable regulator where required, and take remedial action including — where appropriate — deletion of the data from the receiving jurisdiction and re-production within the correct one.

Jurisdiction is the guarantee most likely to matter to a public sector procurement officer, and correspondingly the guarantee most tightly enforced in Archiva's operational architecture.

---

## Non-goals for v1.0

The following are deliberately not covered by v1.0 of this specification. Each is a candidate for later versions or is intentionally out of scope.

- **Cryptographic signing of the manifest itself.** File-level checksums prevent tampering with data files; a signed manifest would prevent tampering with the manifest itself. Planned for **v2.0**.
- **JSON Schema publication for the manifest.** The manifest structure is fully specified in `manifest.md` but is not published as a machine-readable JSON Schema document. Planned for **v1.1**.
- **Immutability enforcement beyond checksums.** Archiva does not use write-once storage or blockchain-anchored proofs of receipt. Consumers who require these can layer them on top of the export using standard tooling.
- **Real-time or delta exports.** v1.0 exports are point-in-time snapshots. Ongoing sync between source and archive is Continuum's territory, not Format's.
- **Long-term producer availability guarantees.** Archiva does not commit that its producer software will remain available in identical form indefinitely. The specification is what remains stable across versions; the software producing exports may evolve.
- **Encryption at rest for delivered exports.** Signed URLs use HTTPS in transit; the archive contents are not additionally encrypted. Customers with sensitive-data requirements can request PGP encryption of the archive at delivery time; this is negotiated per engagement.
- **Guarantees against source system errors.** If the source system contained bad data at snapshot time, the export will contain that bad data. The Format guarantees are about faithful reproduction, not source correction.

## How to raise a guarantee concern

If a customer, third-party consumer, or auditor believes a Format export has violated any of these guarantees, the concern should be raised via the engagement's normal channel or by opening an issue against this specification.

Archiva takes guarantee violations seriously. Every reported concern is investigated and — where the concern is valid — documented, remediated, and used to improve the producer's implementation and this specification.

## Rationale for specific choices

**Why six guarantees, not fewer:** each guarantee addresses a distinct failure mode. Completeness answers *"did you extract everything?"*. Integrity answers *"has anyone tampered with it?"*. Portability answers *"can I read it without you?"*. Reproducibility answers *"can I verify it's what you say it is?"*. Documentation answers *"can I understand it decades from now?"*. Jurisdiction answers *"where does my data live?"*. Collapsing any two would blur the failure mode each is meant to address.

**Why six, not more:** each additional guarantee reduces the memorability and enforceability of the set. Six is where the pattern still fits in a reader's mind. Additional protections (encryption at rest, signing, immutability) belong in optional engagement terms, not in the base specification.

**Why "how a violation would be handled" for every guarantee:** a guarantee without a failure path is more marketing than technical commitment. Being explicit about what would happen if a guarantee were violated makes the guarantee credible to auditors and successor providers.

**Why jurisdiction is a technical guarantee, not a legal one:** legal commitments live in contracts and can be amended. Technical guarantees live in the specification and the infrastructure and are visible to anyone reading the export. For sovereignty to mean anything, it has to be enforced at the technical level, not just promised in contract clauses.

**Why the specification lists what it does *not* guarantee:** honesty compounds. A specification that quietly implies more than it delivers erodes trust when the gap is discovered. A specification that names its own edges builds trust that the covered ground is solid.

## Version history

- **Draft (v1.0)** — Initial specification. Establishes six guarantees, each with technical enforcement, consumer verification, and violation handling. Non-goals explicitly named.
