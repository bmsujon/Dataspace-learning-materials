# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

A folder of Markdown learning materials about the Catena-X / Eclipse Tractus-X dataspace stack. There is no code, build, lint or test tooling. Work here means researching, writing and fact-checking documents.

| File | Purpose | Audience |
| --- | --- | --- |
| `Catena-X & Tractus-X Backend Team Primer.md` | Onboarding primer: ecosystem roles, identity (BPN/DID/VC/VP), DSP and DCP protocols, EDC and IdentityHub architecture, policies, data layer, component map, a hands-on first week, running customer connectors, pitfalls, glossary, ordered reading list | The team's backend engineers, who build Connector as a Service, Dataspace as a Service and passport apps (battery, PCF, textile) |
| `edc-connector-vs-tractusx-edc.md` | Engineering comparison and selection guide: upstream `eclipse-edc/Connector` vs. downstream `eclipse-tractusx/tractusx-edc`, with measured composition, capability matrix, risk register and recommendation | Engineers and decision makers choosing a connector base |
| `dataspace-protocol-guide.md` | Deep dive on the Dataspace Protocol: version history, terminology, every message, state and endpoint, JSON-LD wire format, EDC/Tractus-X implementation details, pitfalls | The same backend engineers, once they need more than the primer's DSP section |
| `decentralized-claims-protocol-guide.md` | Deep dive on the Decentralized Claims Protocol: identity building blocks, self-issued tokens, presentation and issuance protocols, trust model, profiles, EDC/Tractus-X/IdentityHub/IssuerService implementation details, end-to-end trace, debugging | Developers working on connectors, wallets or credentials |

All documents are dated **10 October 2026**, and every version-specific statement is "as of" that date.

`README.md` is the human-facing index. When you add, rename or re-date a document, update its table and "Keeping it current" section.

## Facts shared across documents

The documents overlap heavily. When a fact changes, update it in every file and every section that repeats it. Grep for the old value instead of trusting a single occurrence. The main shared facts are:

- tractusx-edc **0.13.0** (27 Aug 2026) pins upstream **EDC 0.17.0**; `main` is 0.14.0-SNAPSHOT.
- Upstream EDC is in 1.0 release candidates (latest **v1.0.0-RC4**, 29 Sep 2026). No Tractus-X release carries 1.0 yet.
- Tractus-X release train **26.09** (16 Sep 2026) is the first to ship tractusx-edc 0.13.0.
- Catena-X release **CX-Titan** is current; **CX-Neptune** goes live 24 Nov 2026.
- The renames IATP→DCP and DIM→DIV, Management API v3 deprecated in favour of v4 (v5 is virtual-connector/beta), DSP 0.8 vs 2025-1, and the upstream EDR-cache deprecation.
- DSP **2025-1** is Eclipse release 1.0.0 (23 Jul 2025); the current erratum is **2025-1-err2** = release 1.0.2 (10 Sep 2026), editorial only. Tractus-X serves DSP 2025-1 at `/api/v1/dsp/2025-1` and 0.8 at `/api/v1/dsp`.
- DCP **v1.0.1** (12 Nov 2025) is current; 1.0.0 was the Eclipse release of 18 Jul 2025. Tractus-X EDC 0.13.0 requests credentials with the 0.8-era scope alias `org.eclipse.tractusx.vc.type:<Type>:read`; upstream EDC and IdentityHub use `org.eclipse.dspace.dcp.vc.type`. tractusx-identityhub 0.4.1 is built on IdentityHub v0.17.0.

## Conventions per document

### Primer

- Prose is one long line per paragraph. Do not hard-wrap it.
- Claims carry **inline source links** (Catena-X standards, Tractus-X repos and release notes, Eclipse blog posts). New claims need a source link in the same style. The document ends with `Sources for this document are linked inline in each section. As of <date>.`
- Sections are numbered and cross-referenced as "(section N)" about 50 times. Section 1's reading guide also lists section ranges. If you add, remove or reorder sections, renumber every cross-reference and update the reading guide.
- Section 4 has an **"Open decisions for our setup"** table with a Status column the team maintains. Add rows for new open questions rather than burying them in prose. Uncertain statements use the marker "Assumption to confirm:".
- The two `&#91;embedded content: …\]` lines are placeholders for diagrams that were lost when the document was exported. Keep them, and keep their captions in sync with the surrounding text.
- The glossary is section 19 and the ordered reading list is section 20 (phases 1–5). Add new terms and essential links there.

### Comparison guide

- Prose is hard-wrapped at about 100 characters. Tables are not wrapped.
- Every figure must be derived from source trees at a named release tag, git history, or published release notes and decision records (see section 2, "Scope and method"). Judgements and estimates must be labelled as such.
- **Appendix A** lists the shell commands (curl and git against the GitHub repos) used to re-verify figures. Use them, and add any new command you use to derive a number.
- **Appendix D, "Document control"**: any substantive change needs a new version row with the date and a summary of changes. Also update the version line under the title (currently "Version 2.0 · 10 October 2026").
- Appendix B (references) and Appendix C (glossary) should gain entries for any new repos, decision records or abbreviations you introduce.

### DSP guide

- Follows the primer's style: one line per paragraph, inline source links, numbered sections with "(section N)" cross-references, and a closing `Sources ... As of <date>.` line. References into other documents are written as "primer section N" or "comparison guide section N", so renumbering those documents means updating this one too.
- Sections 5–9 restate the DSP specification at tag `2025-1-err2` of `eclipse-dataspace-protocol-base/DataspaceProtocol`. Example messages are copied from the repo's `artifacts/src/main/resources/*/example/` files; check new ones against the JSON Schemas there.
- Section 10 describes EDC v0.17.0 and tractusx-edc 0.13.0 behaviour, with links to the source files at those tags. When the pinned versions change, re-check each linked claim.
- State machines and sequences are Mermaid diagrams, which GitHub renders. Keep transitions identical to the spec's PlantUML sources in `specifications/*/figures/`.

### DCP guide

- Same style and cross-reference rules as the DSP guide; it also references "DSP guide section N".
- Sections 6–10 restate the DCP specification at tag `v1.0.1` of `eclipse-dataspace-dcp/decentralized-claims-protocol`; spec examples come from its `artifacts/src/main/resources/*/example/` files. Examples marked "illustrative" (DID document, credential, token) are constructed, not copied, so keep them consistent with the spec's rules.
- Sections 11–12 describe EDC v0.17.0, tractusx-edc 0.13.0, IdentityHub v0.17.0 and tractusx-identityhub 0.4.1, with source links at those tags. IdentityHub endpoint paths and ports come from the tractusx-identityhub DCP API walkthrough.

## Fact-checking

Both projects change fast, so re-verify any version, date or count against primary sources before you edit it. The primary sources are the GitHub repos and their release tags, the Catena-X standards site (catenax-ev.github.io), the Tractus-X docs and changelog (eclipse-tractusx.github.io), and the Eclipse EDC blog (eclipse-edc.github.io). When you update a fact, also update the "as of" date in the affected document.

## Working rules

- **Commit authorship**: commits must carry **only the user's identity**. Do **not** add
  `Co-Authored-By` / co-author trailers or any AI-attribution to commit messages or PR bodies.
- **Avoid unnecessary fallbacks**, especially for secrets and critical configs. Prefer failing
  fast (explicit error / `fail` in templates) over silently substituting a default value when a
  required secret or critical setting is missing.
- commit only the files related to project. Internal planning files should not be committed.
- PR description should be given in md format
- Keep PR reviews short and to the point so that a human doesn't get bored reading long texts.
- Avoid using emojis. Keep the text as if a human is writing it
