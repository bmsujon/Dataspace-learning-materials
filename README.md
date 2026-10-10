# Dataspace learning materials

Notes for engineers getting up to speed on Catena-X and the Eclipse Tractus-X dataspace stack: how connectors and wallets work, which open-source components to build on, and how they fit together.

## Documents

| Document | What it covers |
| --- | --- |
| [Catena-X & Tractus-X Backend Team Primer](Catena-X%20%26%20Tractus-X%20Backend%20Team%20Primer.md) | The ecosystem and who runs what, identity (BPN, DID, verifiable credentials), the Dataspace Protocol and Decentralized Claims Protocol, EDC and IdentityHub, policies, digital twins, a hands-on first week, running connectors for customers, common pitfalls and a glossary |
| [EDC Connector vs. Tractus-X EDC](edc-connector-vs-tractusx-edc.md) | How upstream Eclipse EDC and Tractus-X EDC relate, what Tractus-X adds and removes, release lag, risks, and which one to build on |
| [Dataspace Protocol (DSP) Learning Guide](dataspace-protocol-guide.md) | DSP in depth: version history, terminology, every message, state and endpoint of the Catalog, Contract Negotiation and Transfer Process protocols, the JSON-LD wire format, how EDC and Tractus-X EDC implement it, and common pitfalls |

## Where to start

- **New to Catena-X:** read the primer. Section 1 tells you which sections to read for what, and section 20 is an ordered reading list of the official specs and docs.
- **Choosing a connector base:** read the executive summary, decision guide and recommendation of the comparison guide (sections 1, 15 and 16).
- **Reading or debugging DSP traffic:** read the primer's DSP overview (section 6), then the DSP guide. Its sections 7 and 8 have the state machines and endpoint tables; section 12 lists common pitfalls.

## Keeping it current

All documents are accurate as of 10 October 2026. Catena-X, Tractus-X, EDC and the DSP specification all release often, so check version numbers and dates against the linked sources before relying on them. Appendix A of the comparison guide has the commands used to verify its figures.

When editing, link a source for every new claim, and update a fact in every document it appears in. [CLAUDE.md](CLAUDE.md) lists the full editing conventions.
