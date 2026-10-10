# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

A folder of Markdown learning materials about the Catena-X / Eclipse Tractus-X dataspace stack. There is no code, build, lint or test tooling. Work here means researching, writing and fact-checking documents.

| File | Purpose | Audience |
| --- | --- | --- |
| `Catena-X & Tractus-X Backend Team Primer.md` | Onboarding primer: ecosystem roles, identity (BPN/DID/VC/VP), DSP and DCP protocols, EDC and IdentityHub architecture, policies, data layer, component map, a hands-on first week, running customer connectors, pitfalls, glossary, ordered reading list | The team's backend engineers, who build Connector as a Service, Dataspace as a Service and passport apps (battery, PCF, textile) |
| `edc-connector-vs-tractusx-edc.md` | Engineering comparison and selection guide: upstream `eclipse-edc/Connector` vs. downstream `eclipse-tractusx/tractusx-edc`, with measured composition, capability matrix, risk register and recommendation | Engineers and decision makers choosing a connector base |

Both documents are dated **10 October 2026**, and every version-specific statement is "as of" that date.

`README.md` is the human-facing index. When you add, rename or re-date a document, update its table and "Keeping it current" section.

## Facts shared across both documents

The documents overlap heavily. When a fact changes, update it in both files and in every section of each file that repeats it. Grep for the old value instead of trusting a single occurrence. The main shared facts are:

- tractusx-edc **0.13.0** (27 Aug 2026) pins upstream **EDC 0.17.0**; `main` is 0.14.0-SNAPSHOT.
- Upstream EDC is in 1.0 release candidates (latest **v1.0.0-RC4**, 29 Sep 2026). No Tractus-X release carries 1.0 yet.
- Tractus-X release train **26.09** (16 Sep 2026) is the first to ship tractusx-edc 0.13.0.
- Catena-X release **CX-Titan** is current; **CX-Neptune** goes live 24 Nov 2026.
- The renames IATP→DCP and DIM→DIV, Management API v3 deprecated in favour of v4 (v5 is virtual-connector/beta), DSP 0.8 vs 2025-1, and the upstream EDR-cache deprecation.

## Conventions per document

### Primer

- Prose is one long line per paragraph. Do not hard-wrap it.
- Claims carry **inline source links** (Catena-X standards, Tractus-X repos and release notes, Eclipse blog posts). New claims need a source link in the same style. The document ends with "Sources for this document are linked inline in each section. As of <date>."
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
