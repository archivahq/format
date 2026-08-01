# Contributing

The Format specification is a commitment Archiva makes to every past, present, and future customer. Every export ever produced against a version of this specification must remain readable and verifiable for as long as the customer holds it. That shapes how we accept changes.

We welcome:

- **Corrections** — factual errors, unclear wording, mismatches between the specification and how Archiva actually produces exports
- **Clarifications** — parts of the specification that leave room for interpretation and could be tightened
- **New examples** — illustrative sample exports that help future readers understand the specification
- **Small improvements** — better structure, better cross-references, better naming

We consider carefully, but do not accept lightly:

- **New formats** (a fifth output alongside CSV / Parquet / JSON / SQL) — every added format is an ongoing engineering and support commitment
- **Manifest additions** — new required fields in the manifest are a breaking change; new optional fields need a clear justification
- **New delivery methods** — the specification currently describes four; each adds surface area
- **Backwards-incompatible changes** — any change that would make a past export unreadable is a major version bump and requires strong grounds

## How to propose a change

**For corrections and clarifications**, open a pull request against the appropriate specification version (usually `spec/draft/`). Explain the problem, propose the fix.

**For substantial changes** — new formats, new manifest fields, new delivery methods, or backwards-incompatible edits — open an issue first. Describe the change, the customer or engineering need behind it, and the alternatives considered. We discuss before drafting.

**For general questions** about the specification, open a discussion or issue. Even questions that don't lead to changes help us find where the specification isn't clear enough.

## What we're looking for in contributions

- **Plain English.** The specification is read by records managers and IT teams, not only by engineers.
- **Precision over comprehensiveness.** Short and exact beats long and hedged.
- **Vendor-neutral.** The specification describes categories and behaviours, not specific tools.
- **Evidence-based.** Claims of "this is how the standard works" should point to the standard.

## Versioning discipline

The specification version reflects the seriousness of the commitment. Please:

- Direct all in-development work at `spec/draft/`
- Do not modify released version folders (`spec/v1.0/`, etc.) except for typo fixes that don't change meaning
- When a released version needs a real change, that's a new version, not an edit to the old one

## Licence

By contributing, you agree that your contribution is released under the same licence as the specification — [Creative Commons Attribution 4.0](./LICENSE) for text, [MIT](./LICENSE-CODE) for any code samples.
