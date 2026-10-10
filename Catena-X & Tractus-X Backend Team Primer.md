# Catena-X & Tractus-X: Backend Team Primer

Oct 10, 2026 · @Md. Wahidul Azam

## 1. Why this doc exists

Every Catena-X participant runs at least two services: a connector that negotiates and moves data, and a wallet that proves who the company is. Tractus-X EDC and Tractus-X IdentityHub are the open-source reference implementations, but the rules are HTTP protocols, so everything around them can be built in any language. This primer explains how those two fit into the wider Catena-X ecosystem, so the team can read specs and code with the right mental model.

The one-paragraph model: companies never upload data to a central platform. Each runs its own connector and wallet. Connectors talk to each other with the **Dataspace Protocol (DSP)** to list offers, agree contracts and start transfers. Before any of that is allowed, the connector asks its wallet to prove identity and membership with the **Decentralized Claims Protocol (DCP)**, built on W3C **DIDs**, **Verifiable Credentials** and **Verifiable Presentations**. A small set of central services (portal, credential issuer, business partner data, discovery) exists only to onboard companies and help them find each other.

Read sections 2–4 for context and where our products fit, 5–7 for identity and the two protocols, 8–10 for the reference connector and wallet, and 11–13 for how policies and data flow end to end. Sections 14–15 map every component and language stack, and 16–17 cover the first hands-on week, onboarding customers and running their connectors. Section 4 ends with the open decisions for our own setup. The ordered reading list is at the end.

## 2. Who is who

Catena-X writes the rules, Tractus-X writes the open-source reference code, and the Eclipse Dataspace Working Group owns the generic protocols underneath both. Keeping these four layers apart avoids most early confusion.

| Layer | Organisation | What it produces | Where we meet it |
| --- | --- | --- | --- |
| Use-case rules | [Catena-X Automotive Network e.V.](https://catenax-ev.github.io/docs/operating-model/who-roles-in-the-catena-x-ecosystem) (association) | Standards (CX-xxxx), operating model, certification, credential issuer definition | Normative MUST/SHOULD our deployment is certified against |
| Reference code | [Eclipse Tractus-X](https://eclipse-tractusx.github.io/) | tractusx-edc, tractusx-identityhub, BDRS, DTR, portal, KITs | The repos we build, extend and deploy |
| Generic dataspace stack | Eclipse Dataspace Working Group | DSP and DCP specs, TCKs, upstream EDC and IdentityHub | Upstream code and specs Tractus-X wraps |
| Origins | IDSA, Gaia-X | Reference architectures, early DSP drafts, compliance/self-descriptions | Background and Gaia-X onboarding checks |

Catena-X ships named releases; the current one is **CX-Titan**, with Saturn, Io and Jupiter before it; CX-Neptune is in preview and goes live on 24 November 2026. Tractus-X ships quarterly calendar releases, the latest being 26.09 (16 September 2026). Component versions such as tractusx-edc 0.11 or 0.13 are mapped to these trains.

### Roles a company can take

A company can hold several roles at once, but only one company may be the CSP-B, which runs the shared identity and discovery services. Every provider role must be certified by a Conformity Assessment Body.

| Role | What it runs | Relevance for us |
| --- | --- | --- |
| Data Provider/Consumer (DPC) | Uses connector + wallet to exchange data | Our end users, or us |
| Enablement Service Provider (ESP) | Hosts connectors, wallets, DTRs for others | Us: Connector and Dataspace as a Service (section 4); needs TISAX Level 2 |
| Business Application Provider (BAP) | Apps built on the dataspace (PCF, traceability) | Us too: the battery, PCF and textile passport apps (section 4) |
| Core Service Provider A (CSP-A) | Marketplace, IAM, semantic hub, BPN discovery; can be run n times | Services we call |
| Core Service Provider B (CSP-B) | Issuer, BDRS, BPDM, connector discovery, custodial wallets; exactly one | The trust anchor today; Cofinity-X fills this role in production |
| Onboarding Service Provider (OSP) | Registration of new participants | Where a new company starts |

Today the CSP-B also temporarily acts as the trust anchor that issues verifiable credentials; the operating model says a dedicated issuer concept is planned. Source: [roles](https://catenax-ev.github.io/docs/operating-model/who-roles-in-the-catena-x-ecosystem).

## 3. Service map: what is central and what is ours

Only onboarding, credential issuance and lookups are central; every company runs its own connector, wallet and data services, and data never passes through a central system ([service map](https://catenax-ev.github.io/docs/operating-model/what-service-map)).

&#91;embedded content: Catena-X service map · core services above, two participants below\]

Core services issue credentials and answer lookups; connectors then talk DSP directly and move data plane to plane. Each connector also calls its own wallet for DCP, and the provider calls the consumer's wallet to verify credentials (section 7).

Other core services you will hear about: the BPDM Gate, Pool and Orchestrator (golden-record company data and BPN issuing), the Semantic Hub (published aspect models), marketplaces, and a legacy OAuth2 identity provider that the operating model says will disappear once all core services use the credential-based access pattern. The connector and the wallet are the mandatory enablement services for every participant; a company can bring its own Catena-X compliant wallet or receive a custodial one at onboarding.

## 4. Our products: where they sit in Catena-X

We build on two layers of the ecosystem. Connector as a Service and Dataspace as a Service host the infrastructure other companies need to take part, and the passport apps (battery, PCF, textile, more to come) run on top of it. In Catena-X terms that makes us an Enablement Service Provider (ESP) and a Business Application Provider (BAP), and both roles need certification ([roles](https://catenax-ev.github.io/docs/operating-model/who-roles-in-the-catena-x-ecosystem)).

| Product | Catena-X role | What we run | Standards and KITs | What we must get right |
| --- | --- | --- | --- | --- |
| Connector as a Service | ESP | A tractusx-edc control plane and data plane per customer, plus its wallet (our IdentityHub or the customer's own) | CX-0018, CX-0149, CX-0152; Connector KIT | Tenant isolation, fleet upgrades, partners on both DSP 0.8 and 2025-1, custody of customer keys |
| Dataspace as a Service: participant stack | ESP | Connector, wallet, DTR and submodel server per customer | The above plus CX-0002 and CX-0003; Digital Twin KIT | The above plus twin registration and aspect-model versions |
| Dataspace as a Service: whole dataspace | Operator of a dataspace outside Catena-X | Issuer, onboarding, BPN/DID directory and discovery, plus participant stacks | DSP and DCP; our own credentials, policy profile and rulebook | Governance: who may issue which credential and how members join |
| Battery passport | BAP | Passport twins and submodels in the customer's DTR, offered through their connector | [CX-0160](https://catenax-ev.github.io/docs/next/standards/CX-0160-BatteryPassport/CX-0160-BatteryPassport-base) (preview), [CX-0143](https://catenax-ev.github.io/docs/standards/CX-0143-UseCaseCircularEconomyDigitalProductPassportStandard/introduction), [Eco Pass KIT](https://eclipse-tractusx.github.io/docs-kits/kits/eco-pass-kit/adoption-view); EU Battery Regulation | The 18 February 2027 deadline and a data-model migration |
| PCF | BAP | PCF submodels for synchronous reads and a PCF exchange endpoint for asynchronous requests | [CX-0136](https://catenax-ev.github.io/docs/standards/CX-0136-UseCasePCF), [PCF Rulebook](https://catenax-ev.github.io/docs/rulebooks/CX-NFR-PCF), [PCF Exchange KIT](https://eclipse-tractusx.github.io/docs-kits/kits/product-carbon-footprint-exchange-kit/adoption-view), [PCF Data Acquisition KIT](https://eclipse-tractusx.github.io/docs-kits/kits/pcf-data-acquisition-kit/adoption-view) | The asynchronous flow and frequent aspect-model versions |
| Textile passport | None yet: Catena-X is automotive | The same app pattern, in a dataspace we operate | EU ESPR; the CX-0143 generic passport modules as a template | Rules that are still being written |

Assumption to confirm: Dataspace as a Service covers both hosting participant stacks inside Catena-X and operating whole dataspaces for industries that have none, such as textile.

### Connector and Dataspace as a Service

- Every customer is its own participant, with its own BPN, DID, credentials and keys. We host the identity; the customer owns it. did:web lets us serve each customer's DID document from our domain, e.g. did:web:wallet.ours.example:acme.
- tractusx-edc runs one participant per deployment. Connector as a Service therefore means one control plane, data plane, database and Vault path per customer. IdentityHub is already multi-tenant: one deployment holds a participant context per customer (section 10).
- Upstream EDC 1.0 adds the Virtual Connector, which runs many participants in one runtime with Management API v5 paths per participant. No Tractus-X release ships it yet (section 8).
- tractusx-edc is splitting into a dataspace-agnostic core and per-dataspace flavours such as catena-x, factory-x and construct-x ([decision record](https://github.com/eclipse-tractusx/tractusx-edc/blob/main/docs/development/decision-records/2026-01-07_multi-dataspace-support/README.md)). That split is what lets the same code serve dataspaces outside Catena-X.
- Operating a whole dataspace means running its trust anchor: an IssuerService that issues membership credentials, plus a directory and discovery. The IdentityHub [DCP API walkthrough](https://github.com/eclipse-tractusx/tractusx-identityhub/blob/main/docs/usage/dcp-api-walkthrough/README.md) issues a MembershipCredential end to end from your own issuer, which is the core of that product.
- The umbrella chart's [Hausanschluss bundles](https://github.com/eclipse-tractusx/tractus-x-umbrella/blob/main/docs/user/common/guides/hausanschluss-bundles.md) package the enablement services as independently deployable bundles, a useful reference for per-customer deployments. Section 17 covers running them.

### Passport applications

- Every passport follows one pattern: a digital twin in the provider's DTR, the passport as a submodel defined by a SAMM aspect model, offered as an EDC asset, and read through discovery, DTR lookup, negotiation and a pull (section 13). Passport offers use the usage purpose cx.circular.dpp:1 ([CX-0143 API section](https://catenax-ev.github.io/docs/standards/CX-0143-UseCaseCircularEconomyDigitalProductPassportStandard/CX-0143-API)).
- Catena-X covers only the access-controlled exchange between companies; CX-0143 explicitly leaves public interfaces out. The public part of a passport (the QR code on the product, data for consumers and authorities) needs its own path in our apps, and the EU's passport registry opened on 20 July 2026 ([Battery-Tech](https://battery-tech.net/why-the-eu-is-about-to-miss-its-own-battery-passport-deadline-while-industrys-stays-fixed/)).
- **Battery.** The [EU Battery Regulation](https://eur-lex.europa.eu/eli/reg/2023/1542/oj) requires a passport from 18 February 2027 for EV, light-means-of-transport and industrial batteries above 2 kWh. The implementing act on who may see which data was still pending in August 2026, with the Commission targeting Q4 2026 ([Battery-Tech](https://battery-tech.net/why-the-eu-is-about-to-miss-its-own-battery-passport-deadline-while-industrys-stays-fixed/)).
- In Catena-X, [CX-0160](https://catenax-ev.github.io/docs/next/standards/CX-0160-BatteryPassport/CX-0160-BatteryPassport-base) adopts the joint Catena-X/IDTA battery passport model (IDTA-02035 parts 1–7, aligned with DIN DKE SPEC 99100). It belongs to CX-Neptune, which goes live on 24 November 2026, and is not certifiable before then. The 26.09 [Eco Pass KIT](https://eclipse-tractusx.github.io/docs-kits/kits/eco-pass-kit/adoption-view) still documents the older Catena-X Battery Pass model 6.0.0, so plan the migration.
- **PCF.** [CX-0136](https://catenax-ev.github.io/docs/standards/CX-0136-UseCasePCF) allows two styles. Synchronous: the consumer reads a PCF submodel found through the DTR. Asynchronous: the consumer calls the provider's PCF exchange asset with GET /productIds/{id}?requestId=…, and the provider answers with a PUT to the consumer's own exchange asset, so both sides need a connector and an offer.
- PCF exchange offers use dct:type cx-taxo:PcfExchange and the usage purpose cx.pcf.base:1. The current release specifies aspect model io.catenax.pcf 9.0.0, and Tractus-X 26.09 already published 10.0.0 ([26.09 changelog](https://eclipse-tractusx.github.io/blog-changelog/release-26-09)). Calculation rules come from the [PCF Rulebook](https://catenax-ev.github.io/docs/rulebooks/CX-NFR-PCF).
- **Textile.** Catena-X has no textile standard. Under the EU [Ecodesign regulation (ESPR)](https://eur-lex.europa.eu/eli/reg/2024/1781/oj) the textile delegated act is expected around 2027 and would apply about 18 months later, so mandatory textile passports are likely from 2028 at the earliest ([cosh.eco](https://cosh.eco/en/articles/eu-digital-product-passport-espr-explained-fashion-textiles)).
- CX-0143's generic passport model (urn:samm:io.catenax.generic.digital\_product\_passport:6.1.0) is a set of reusable modules aligned with ESPR. Catena-X forbids exchanging it as-is, but its modules are a sensible start for a textile model in a dataspace we run ([CX-0143 aspect models](https://catenax-ev.github.io/docs/standards/CX-0143-UseCaseCircularEconomyDigitalProductPassportStandard/CX-0143-SEM)).

### Open decisions for our setup

Our setup is still being worked out by trial and error. These are the decisions this doc raises; set Status as the team tries and settles each one, and add rows as new questions come up.

| Decision | Options | Where this doc helps | Status |
| --- | --- | --- | --- |
| Wallet for each customer | Our own IdentityHub (multi-tenant), or the operator's DIV wallet | Tractus-X 26.09 still relies on the DIV bridge (sections 10, 18) | Open |
| Tenancy model for Connector as a Service | One tractusx-edc deployment per customer now, or wait for the Virtual Connector | Sections 4, 8 | Open |
| Scope of Dataspace as a Service | Participant stacks inside Catena-X only, or also whole dataspaces such as textile | Section 4 | Open |
| Catena-X release to target | CX-Titan now, or CX-Neptune from 24 November 2026, which the battery standard CX-0160 needs | Sections 2, 4 | Open |
| Battery data model | Catena-X Battery Pass 6.0.0, or the joint IDTA model from CX-0160 | Section 4 | Open |
| DSP versions to support | 2025-1 only, or also 0.8 for Jupiter-era partners | Sections 9, 18 | Open |
| Management API version in our backends | v3 as in the Tractus-X docs, or v4 with v3 only for Tractus-X endpoints | Sections 15, 16 | Open |
| Learning when transfers finish | Callbacks, polling or the EDR API | Section 16 | Open |
| Digital twin registry | Tractus-X DTR, or another AAS registry that meets the CX-0002 profiles | Sections 12, 14 | Open |
| Public passport path | How QR codes, public views and the EU passport registry sit beside Catena-X | Section 4 | Open |
| Onboarding route for customers | Through the operator's OSPs, or becoming a certified OSP ourselves | Section 17 | Open |
| Telemetry stack | OpenTelemetry collector plus which storage and dashboards | Section 17 | Open |

## 5. Identity foundations: BPN, DID, VC, VP

A company in Catena-X has two identifiers: a business one (the BPN) and a cryptographic one (the DID). Credentials bind the two together, and presentations prove the credentials belong to whoever is calling.

### BPN: the business identifier

The Business Partner Number is issued by the BPDM services and never changes. A BPNL identifies a legal entity, a BPNS a site (a plant), and a BPNA an address. The BPNL is the legally binding identifier used in data exchange contracts, so most Catena-X policies were historically written against BPNLs ([service map](https://catenax-ev.github.io/docs/operating-model/what-service-map)).

### DID: the cryptographic identifier

A DID such as `did:web:wallet.example.com:acme` resolves to a DID document holding two things: public keys (verification methods) and service endpoints. For `did:web` the resolver simply fetches `https://wallet.example.com/acme/did.json` over HTTPS, so there is no blockchain and no central registry. The service list includes the company's **Credential Service** endpoint, which is how a verifier finds where to ask for credentials. The IdentityHub generates and publishes this document for us.

### VC: a signed claim

A Verifiable Credential is a JSON document (W3C VC Data Model) signed by an issuer. It names an issuer DID, a subject DID, an expiry, a type and a set of claims. Anyone can verify it offline by resolving the issuer DID and checking the signature; the issuer is never called. Revocation works through a status list credential the issuer publishes, which verifiers fetch and check.

The Catena-X credentials we will see in every request. They were introduced in CX-0149 v1.0.0; current schemas live in [CX-0050 Catena-X Specific Credentials](https://catenax-ev.github.io/docs/standards/CX-0050-CXSpecificCredentials/CX-0050-CatenaXSpecificVerifiableCredentials), and wallet rules in [CX-0149 Wallet Requirements v2.0.0](https://catenax-ev.github.io/docs/standards/CX-0149-WalletRequirements):

| Credential | Who holds it | What it claims | Typical use in policies |
| --- | --- | --- | --- |
| MembershipCredential | Every participant, issued at onboarding | Holder is onboarded and accepted Catena-X terms | Gate on any catalog or contract |
| BpnCredential | Every participant, issued at onboarding | Holder DID owns BPNL x | Map a DID to a BPN; BPN-based access |
| Framework agreement / DataExchangeGovernanceCredential | Participants who signed that framework agreement | Holder signed a use-case framework agreement (CX-0050) | `FrameworkAgreement` constraint in contract policies |
| Use-case credentials (e.g. DismantlerCredential) | Participants in that role, e.g. dismantlers | Role-specific attributes | Restrict data to a role |

Under [CX-0149 v1.0.0](https://catenax-ev.github.io/docs/Jupiter/standards/CX-0149-Dataspaceidentityandidentification) (now only in the archived CX-Jupiter release; the current release replaced it with Wallet Requirements v2.0.0), credentials expired after at most 12 months, and issuers had to support revocation by issuer, revocation by holder and expiry; connectors must reject invalid credentials in any negotiation. Check CX-0050 for the current validity rules before relying on these numbers.

### VP: proof of possession

A Verifiable Presentation wraps one or more VCs and is signed by the **holder**, not the issuer. That second signature proves the presenter controls the subject DID, so a stolen VC alone is useless. In our stack the IdentityHub creates VPs on request; the EDC only verifies them.

### Three roles to keep straight

- **Issuer**: signs VCs. Today that is the CSP-B's issuer service; the association defines who may issue.
- **Holder**: stores VCs and signs VPs. That is our IdentityHub.
- **Verifier**: checks VPs before allowing something. That is the provider's EDC.

## 6. Dataspace Protocol (DSP): how connectors talk

DSP is three small, asynchronous, JSON-LD-over-HTTPS protocols between two connectors: Catalog, Contract Negotiation and Transfer Process. It governs *access to* data but never carries the data itself. The current release is [DSP 2025-1](https://eclipse-dataspace-protocol-base.github.io/DataspaceProtocol/2025-1/) (Eclipse spec 1.0.0, July 2025); Catena-X still also runs the older 0.8 dialect (see section 18).

### Catalog protocol

The consumer sends a `CatalogRequestMessage` (optionally with a filter) and gets back a DCAT catalog. Each dataset carries one or more ODRL offers (`odrl:hasPolicy`) and the distributions (transfer types) it supports, e.g. `HttpData-PULL`. The provider filters the catalog by the caller's credentials, so two consumers can see different catalogs. That filtering is driven by the **access policy** in EDC (section 11).

### Contract negotiation

Both sides keep their own copy of a negotiation state machine and move it with messages. The happy path:

1. Consumer sends `ContractRequestMessage` with the offer it picked → **REQUESTED**.
2. Provider may counter with `ContractOfferMessage` → **OFFERED** (EDC providers usually skip this).
3. Consumer may send an `accepted` event → **ACCEPTED**.
4. Provider sends `ContractAgreementMessage` containing the signed agreement → **AGREED**.
5. Consumer sends `ContractAgreementVerificationMessage` → **VERIFIED**.
6. Provider sends a `finalized` event → **FINALIZED**. The agreement id is now usable.

Either side can send `ContractNegotiationTerminationMessage` at any point → **TERMINATED**. Every message is answered with an HTTP 200/201 ack; the real response arrives later as a callback to the other side's DSP endpoint. That is why EDC models negotiations as persisted state machines rather than request/response calls.

### Transfer process

With an agreement id, the consumer sends `TransferRequestMessage` naming the transfer type → **REQUESTED**. The provider answers `TransferStartMessage` → **STARTED**; for a pull transfer this message carries a `dataAddress` with an endpoint and short-lived token. The transfer can then be **SUSPENDED** and resumed, and ends in **COMPLETED** or **TERMINATED**.

- **Pull**: the consumer calls the provider's data plane with the token from the `dataAddress`. This is the normal Catena-X pattern (`HttpData-PULL`).
- **Push**: the provider's data plane writes to a destination the consumer named (an S3 bucket, an HTTP sink).

### What DSP deliberately leaves out

- **Identity**: messages carry an `Authorization` header, but its format and validation are delegated to an identity protocol. In Catena-X that is DCP (next section).
- **The data wire protocol**: actual bytes flow over HTTP, S3, Kafka, etc., handled by the data plane.
- **Policy vocabularies**: DSP uses ODRL but each dataspace defines the terms (Catena-X does so in its policy standard).
- **Discovery and versions**: a connector advertises which DSP versions it speaks at `/.well-known/dspace-version`, which lets a 2025-1 connector and an 0.8 connector find a common dialect.

## 7. Decentralized Claims Protocol (DCP): how connectors prove identity

DCP fills the identity hole DSP leaves open: it defines what goes in the `Authorization` header of every DSP call and how the receiver checks it. It has three parts, and the [current release is v1.0.1](https://eclipse-dataspace-dcp.github.io/decentralized-claims-protocol/v1.0.1). DCP started inside Tractus-X under the name IATP, which is why older code and config still say `iatp`.

| Part | Question it answers | Who talks to whom |
| --- | --- | --- |
| Base Identity Protocol | How do I sign a token that says "I am did:x"? | Connector ↔ its own Secure Token Service (STS) |
| Verifiable Presentation Protocol | How does a verifier fetch my credentials? | Provider EDC → consumer's Credential Service |
| Credential Issuance Protocol | How do I obtain credentials from an issuer? | Holder's IdentityHub ↔ Issuer Service |

### Presentation flow (runs on every DSP request)

1. The consumer EDC asks its STS for a **self-issued ID token**: a JWT with `iss` = `sub` = its own DID, `aud` = the provider's DID, a short expiry and a `jti`.
2. The STS embeds a second, opaque **access token** in the JWT's `token` claim. It is scoped to the credentials the consumer is willing to show, e.g. `org.eclipse.tractusx.vc.type:MembershipCredential:read` in Tractus-X EDC (upstream EDC and IdentityHub use the alias `org.eclipse.dspace.dcp.vc.type`).
3. The consumer sends the DSP message with that JWT as its `Authorization` header.
4. The provider EDC resolves the consumer's DID, checks the JWT signature against the DID document's keys, and reads the `CredentialService` service entry.
5. The provider asks its own STS for a self-issued token addressed to the consumer, with the consumer's access token in its `token` claim, and sends a `PresentationQueryMessage` to `<credentialService>/presentations/query` with that token as its bearer token, listing the scopes its policies need.
6. The consumer's IdentityHub validates the access token, selects matching VCs, signs a VP and returns it.
7. The provider verifies the VP signature, each VC's issuer signature, expiry and revocation status, and checks each issuer against its configured **trusted issuers**. The VC claims become the input for policy evaluation (section 11).

Only the provider ever sees the consumer's credentials, and only the types it asked for. No central server sees who talks to whom.

### Scopes and why we configure them

The consumer must decide *before* the call which credential types to expose, because the access token is minted first. EDC derives scopes from policies it is about to evaluate, and Tractus-X EDC adds always-on default scopes. Configuration looks like `tx.edc.iam.dcp.default-scopes.<alias>.type=MembershipCredential` with `.alias` and `.operation` alongside ([tractusx-edc extensions](https://deepwiki.com/eclipse-tractusx/tractusx-edc/4.2-extensions)). Versions before 0.13.0 used the prefix tx.edc.iam.iatp. A missing scope is one of the most common causes of a silently empty catalog.

### Issuance flow (runs at onboarding and renewal)

1. The holder's IdentityHub sends a `CredentialRequestMessage` to the issuer's endpoint, found through the issuer's DID document, authenticated with a self-issued token.
2. The issuer checks its attestation sources (for Catena-X: the onboarding record and BPN), then signs the VC.
3. The issuer delivers it asynchronously with a `CredentialMessage` to the holder's Credential Service storage endpoint.
4. The issuer publishes a revocation status list; renewals repeat the same exchange before expiry.

Issuers can also push credentials with a `CredentialOfferMessage`. Catena-X production does not use this DCP issuance flow yet: the CSP-B's issuer component (ssi-credential-issuer) writes credentials into the participant's DIV wallet through SAP's SSI DIM Middle Layer. The DCP-native IssuerService already ships in the tractusx-identityhub repository (section 10).

### Trust model in one line

A verifier trusts a credential only if its issuer DID is in the verifier's trusted-issuer list. Changing which issuers a connector trusts is a configuration change, not a code change, and it is the main knob for multi-issuer or cross-dataspace setups.

## 8. EDC architecture: the reference connector

EDC is not a framework you call; it is a plug-in runtime you assemble. A "connector" is a Gradle distribution of extension modules, split into a **control plane** that speaks DSP and evaluates policies, and a **data plane** that moves bytes ([adopters manual](https://eclipse-edc.github.io/documentation/for-adopters/)).

Everyone should understand the concepts in this section. The extension details matter only to people changing EDC itself, which requires the JVM; applications in any language integrate through EDC's HTTP APIs (section 15).

### The extension model

- An extension is a class implementing `ServiceExtension`, registered through a `META-INF/services` provider file (plain `java.util.ServiceLoader`) and added to the runtime with `runtimeOnly(project(...))` ([extension model](https://eclipse-edc.github.io/documentation/for-contributors/runtime/extension-model/)).
- Dependency injection is EDC's own, not Spring: `@Inject` fields, `@Provider` factory methods, `@Provides`/`@Requires` to declare the graph, and `@Setting` for configuration. The runtime topologically sorts extensions from these annotations.
- Extensions should only read config and wire services; business logic belongs in services behind SPI interfaces. Modules come in pairs: `*-spi` (interfaces, types) and the implementation.
- Config resolves from a config file, then environment variables (`edc.foo.bar` → `EDC_FOO_BAR`), then `-D` system properties. Secrets never go in config; they go in a `Vault`.
- `./gradlew autodoc` emits an `edc.json` listing every setting, injected service and extension point. It is the fastest way to answer "what does this module need".

### Control plane: the domain model

| Entity | Owner | Purpose |
| --- | --- | --- |
| Asset | Provider | Data offering with metadata and a `dataAddress` pointing at the real backend |
| PolicyDefinition | Provider | A named ODRL policy |
| ContractDefinition | Provider | Binds an asset selector to an **access policy** (who sees it in the catalog) and a **contract policy** (who may sign) |
| ContractNegotiation | Both | Persisted DSP negotiation state machine |
| ContractAgreement | Both | The signed result; referenced by every transfer |
| TransferProcess | Both | Persisted DSP transfer state machine |
| EDR (Endpoint Data Reference) | Consumer | Endpoint + token for pulling data under an agreement |

Negotiations and transfers are processed by state-machine loops that pick up entities from a store, advance them one step and persist them. Stores are in-memory for tests or PostgreSQL in production, and entities are leased so several control-plane replicas can share one database. When debugging, look at the entity's `state` and `errorDetail` in the database before reading logs.

### APIs a connector exposes

| API | Audience | Notes |
| --- | --- | --- |
| Management API | Our own backends | v3 has been stable \~2 years. v4 aligns with DSP 2025-1, validates payloads with JSON Schema, and became stable in EDC 0.17.0, which also deprecated v3 and put v5 in beta; v3 is removed no earlier than three months after deprecation ([blog](https://eclipse-edc.github.io/blog/2026/02/19/management-api-upcoming-versions/)) |
| Protocol (DSP) API | Other connectors | Must be internet-reachable; this is where DCP tokens arrive |
| Data plane public API | Consumers pulling data | Validates the EDR token, proxies to the backend |
| Signaling / control API | Control plane ↔ data plane | Internal only |

v5, in beta since EDC 0.17.0, adds a `participantContextId` to API paths so one runtime can host many participants (the "Virtual Connector", EDC-V); no date for a stable v5 was set at the time of writing.

### Data plane and the Data Plane Signaling shift

Today EDC ships a Data Plane Framework (DPF) with HTTP, S3, Kafka and other transfer types. Upstream has decided to replace the control-to-data-plane interface with the [Data Plane Signaling](https://eclipse-edc.github.io/blog/2026/01/19/data-plane-signaling/) specification and to deprecate DPF, with removal eligible two releases after deprecation. Under the new model the data plane, not the control plane, creates the data address, and any application can become a data plane by implementing the spec; SDKs exist for Java, Go, Rust and .NET. tractusx-edc 0.13.0 already ships upstream 0.17.0 with management API v4 and a Data Plane Signaling implementation ([release notes](https://github.com/eclipse-tractusx/tractusx-edc/releases/tag/0.13.0)).

### Version 1.0

Upstream published [EDC 1.0.0-RC1](https://eclipse-edc.github.io/blog/2026/09/07/eclipse-dataspace-components-1.0.0-rc1-released/) on 7 September 2026 and had tagged release candidates up to RC4 by 10 October 2026. Version 1 marks functional completeness (DSP and DCP support, Data Plane Signaling, EDC-V as the default capability set, management API v4/v5) but is explicitly **not** an LTS release; the normal deprecation policy still applies ([path to v1](https://eclipse-edc.github.io/blog/2026/03/06/the-path-to-version-1-for-the-eclipse-dataspace-components/)). Tractus-X will adopt it in a later tractusx-edc release, so plan upgrades rather than pinning forever.

## 9. Tractus-X EDC: what it adds to upstream EDC

[tractusx-edc](https://github.com/eclipse-tractusx/tractusx-edc) is upstream EDC plus Catena-X extensions, packaged as Docker images and Helm charts. If you extend the connector, most of your code reading will be in its `edc-extensions/` folder; the core state machines and DSP/DCP implementations live upstream.

### Distributions

| Distribution | Contains | Use |
| --- | --- | --- |
| `edc-controlplane-postgresql-hashicorp-vault` | Control plane, PostgreSQL stores, HashiCorp Vault secrets | Production |
| `edc-dataplane-hashicorp-vault` | Data plane, Vault | Production |
| `edc-runtime-memory` | Everything in one process, in-memory stores | Local development and tests only |

The repo's README notes a known incompatibility with HashiCorp Vault 1.18.1; check it before choosing a Vault version.

### Catena-X extensions worth knowing

| Extension | What it does | Why it exists |
| --- | --- | --- |
| `tx-dcp` | DCP wiring, default credential scopes, STS client config | Every DSP call needs DCP; renamed from IATP in 0.13.0 |
| `bdrs-client` | Resolves BPN ↔ DID against the central BDRS, with caching | Catena-X APIs and policies still address partners by BPN |
| `bpn-validation` | `BusinessPartnerGroup` policy function with `isAnyOf`, `isAllOf`, `isNoneOf`; groups stored in the connector | Scalable partner allow-lists instead of hard-coded BPNs |
| `cx-policy` | Catena-X constraints such as `FrameworkAgreement`, `UsagePurpose`, `Membership` | Implements the Catena-X ODRL profile (section 11) |
| `agreements` | Agreement retirement API and store | Lets a provider revoke a signed agreement early |
| `edr-api-v2`, `edr-core` | EDR cache and management endpoints, with automatic refresh | Consumer apps fetch a ready-to-use token instead of re-negotiating |
| `dataplane-token-refresh` | OAuth2-style refresh of data plane access tokens | Pull tokens are short-lived (300 s default) |
| `dataplane-proxy-consumer-api` | Consumer-side proxy that looks up the EDR and calls the provider | Apps call one local URL; the proxy handles tokens |
| `migrations` | Flyway-style PostgreSQL schema migrations | Upgrades without manual SQL |

Sources: [extension overview](https://deepwiki.com/eclipse-tractusx/tractusx-edc/4.2-extensions), [0.13.0 release](https://github.com/eclipse-tractusx/tractusx-edc/releases/tag/0.13.0). The overview reflects an older commit; 0.13.0 removed the federated catalog extension and added an optional Kafka streaming extension.

### Important behaviours in recent releases

- **Partner identity**: it depends on the DSP version. Over DSP 0.8 a partner is its BPN, taken from the holderIdentifier of its MembershipCredential; over DSP 2025-1 it is its DID, which BusinessPartnerGroup policies resolve to a BPN through BDRS. The BusinessPartnerNumber constraint reads the bpn claim of the BpnCredential, so that credential is still required.
- **DSP 0.8 is now Tractus-X code**: upstream dropped the 0.8 dialect, so tractusx-edc carries its own implementation to talk to older Catena-X connectors.
- **Management API v4 and Data Plane Signaling** arrived with upstream 0.17.0 in tractusx-edc 0.13.0.
- **Wallet naming**: `dim` became `div` in config, and `iatp` became `dcp`; old keys in Helm values are a common upgrade break.

### Where its docs live

The repo's `docs/` folder holds a management API walkthrough (assets, policies, contract definitions, catalog, negotiations, transfers, EDRs, agreements) and one migration guide per version jump. Read the walkthrough once end to end; read migration guides only for the versions you cross.

## 10. IdentityHub and Issuer Service

IdentityHub is a machine-to-machine wallet: it owns a participant's keys, DID document and credentials, signs self-issued tokens and builds presentations. It is built on the same EDC runtime and extension model as the connector, so everything in section 8 applies ([IdentityHub manual](https://eclipse-edc.github.io/documentation/for-adopters/identity-hub/)). It implements DCP, not OpenID for Verifiable Credentials, because OID4VC assumes a human in the loop.

[tractusx-identityhub](https://github.com/eclipse-tractusx/tractusx-identityhub) is the Tractus-X distribution of upstream IdentityHub; release 0.4.1 ships in Tractus-X 26.09, and its main branch already tracks EDC 0.18.0. It ships two components, each with a production chart (PostgreSQL + HashiCorp Vault) and a memory-only chart for demos:

- **IdentityHub** (`tractusx-identityhub`): the holder's wallet. Stores VCs, manages DIDs and key pairs, answers `/presentations/query`, issues self-issued ID tokens.
- **IssuerService** (`tractusx-issuerservice`): issues VCs from credential definitions and attestation rules, and manages revocation.

The README still describes the project as under heavy development and recommends the localhost install guide for the most current experience.

### Resources it manages

| Resource | States | What to know |
| --- | --- | --- |
| Participant context | CREATED → ACTIVATED ⇄ DEACTIVATED | The tenant boundary. One per identity (usually one per company); deleting it cascades to everything below |
| DID document | published / unpublished | Generated on activation; served under `/did` by the built-in `did:web` publisher, which is not cached, so put a web server or CDN in front at scale |
| Key pair | CREATED, ACTIVATED, ROTATED, REVOKED | Activation adds the public key to the DID document; rotation keeps the old public key until signed artefacts expire; revocation removes it immediately |
| Verifiable credential | stored | Arrives through DCP issuance or the admin API; scopes map to VC types when building VPs |

Private keys live in HashiCorp Vault. IdentityHub can also delegate signing to Vault's Transit engine so the private key never enters JVM memory, which is worth enabling in production.

### Its APIs

- **Identity API** (admin): CRUD for participant contexts, keys, DIDs and credentials. It must never be internet-facing; each participant context gets its own API key.
- **DCP API** (public): the presentation query and credential storage endpoints other parties call.
- **STS**: issues self-issued ID tokens for our connector. It can be embedded in IdentityHub or run separately.
- **DID endpoint** (public): serves `did.json` for `did:web` resolution.

### Deployment shapes

- **Organisation-wide**: one clustered IdentityHub serving all connectors and dataspaces of a company. Best for central key governance.
- **Per dataspace**: one IdentityHub per dataspace, for regional isolation.
- **Embedded**: in the same JVM as the connector, for small or edge setups.

### Presentation details worth knowing

VPs are JWT-encoded by default, because the Linked Data Proof format measured roughly an order of magnitude slower. Queries use scopes; DIF Presentation Exchange is not yet supported. Credentials are fetched by a separate call rather than sent in a header because VPs exceed proxy header limits, and because a provider's policy monitor can re-query a consumer's credentials during a long-running transfer without the consumer initiating anything.

## 11. Policies: where credentials turn into contracts

In Catena-X a policy is both an access-control rule and a legally binding contract clause. Every data offer is exactly one dataset plus one **access policy** plus one **usage policy**; once the consumer accepts it, the usage policy becomes the terms of the agreement ([CX-0152](https://catenax-ev.github.io/docs/standards/CX-0152-PolicyConstrainsForDataExchange)).

### How EDC evaluates policies

EDC's policy engine evaluates ODRL constraints in named **scopes**, and each scope runs at a different moment:

| Scope | When it runs | Usually checks |
| --- | --- | --- |
| `catalog` | Consumer requests the catalog | Access policy: may this caller even see the offer? |
| `contract.negotiation` | Consumer requests a contract | Contract/usage policy: may this caller sign? |
| `transfer.process` | Consumer starts a transfer | Is the agreement still valid for this caller? |
| `policy.monitor` | Periodically during long transfers | Re-checks credentials and dates; can terminate the transfer |

Each ODRL `leftOperand` maps to a Java function (an `AtomicConstraintFunction` or similar) registered for one or more scopes. The function receives the constraint's operator and rightOperand plus the caller's verified claims from DCP, and returns true or false. Adding a new Catena-X constraint therefore means writing one function and registering it; a constraint with no registered function fails evaluation. EDC docs describe this as a code-first approach ([adopters manual](https://eclipse-edc.github.io/documentation/for-adopters/)).

### Catena-X rules (CX-0152)

- Only constraints listed in the Catena-X policy schema may be used, with only their allowed operators and values. Multiple constraints are chained with `odrl:and`.
- Every policy must declare the ODRL `profile` `https://w3id.org/catenax/2025/9/policy/`.
- The provider must always verify the **MembershipCredential**, even if the access policy does not list it.
- Every usage policy visible in a catalog must contain a **FrameworkAgreement** constraint and at least one **UsagePurpose** constraint. A **ContractReference** can point at an existing bilateral contract.

A caution on the profile rule: neither the CX-0152 appendix samples nor the tractusx-edc walkthrough show a profile property, and the examples in this doc follow them. Confirm what your tractusx-edc version and your certification expect before relying on either.

Which credential backs which constraint:

| Constraint | Validated against | Typical place |
| --- | --- | --- |
| `Membership` | MembershipCredential | Access policy (always checked anyway) |
| `BusinessPartnerNumber` / `BusinessPartnerGroup` | BpnCredential for BusinessPartnerNumber; for BusinessPartnerGroup, the partner's DID resolved through BDRS (or its MembershipCredential BPN over DSP 0.8) | Access policy |
| `FrameworkAgreement` | DataExchangeGovernanceCredential | Usage policy (mandatory) |
| `UsagePurpose` | Not a credential; a contractual purpose such as a use-case code | Usage policy (mandatory) |
| Dates, durations, affiliates, warranty, liability, jurisdiction, confidentiality | JSON Schema validation of the value | Usage policy (optional) |

CX-0152 lists 24 usage constraints in a recommended contract order, from framework agreement reference through usage purpose, data-usage end dates, warranty and liability to jurisdiction. Each constraint's schema carries a `$comment` explaining its legal meaning, which is worth reading once.

### A minimal usage policy for a passport

```json
{
  "@context": [
    "https://w3id.org/catenax/2025/9/policy/odrl.jsonld",
    "https://w3id.org/catenax/2025/9/policy/context.jsonld"
  ],
  "@type": "Set",
  "@id": "dpp-usage-policy",
  "permission": [{
    "action": "use",
    "constraint": [{ "and": [
      { "leftOperand": "FrameworkAgreement", "operator": "eq", "rightOperand": "DataExchangeGovernance:1.0" },
      { "leftOperand": "UsagePurpose", "operator": "isAnyOf", "rightOperand": ["cx.circular.dpp:1"] }
    ]}]
  }]
}
```

It follows the usage-policy sample in the CX-0152 appendix, with the passport usage purpose from CX-0143. Take other terms and allowed values from the schema samples in the [CX-0152 appendix](https://catenax-ev.github.io/docs/standards/CX-0152-PolicyConstrainsForDataExchange/Appendix) before using it.

### Practical consequences for our team

- A consumer whose catalog is empty usually failed the **access policy**: missing credential, missing DCP scope, or an untrusted issuer.
- A negotiation that ends in TERMINATED usually failed the **usage policy**: an unknown constraint, a wrong rightOperand, or a missing framework-agreement credential.
- Policies are compared structurally during negotiation, so the consumer must send back the offer exactly as published. Re-serialising JSON-LD carelessly is a classic cause of rejected negotiations.

## 12. The data layer: digital twins, aspect models and discovery

EDC moves bytes but does not say what they mean. Catena-X adds a semantic layer on top: data is organised as **digital twins** in the Asset Administration Shell (AAS) format, each twin's data comes in **submodels** whose structure is defined by **SAMM aspect models**, and a per-provider **Digital Twin Registry (DTR)** acts as the address book ([CX-0002 v2.4.0](https://catenax-ev.github.io/docs/standards/CX-0002-DigitalTwinsInCatenaX)).

### The pieces

| Piece | What it is | Standard / implementation |
| --- | --- | --- |
| Digital twin (AAS shell) | One entry per real asset (a part type, a serialised part, a vehicle), with IDs and a list of submodel descriptors | IDTA AAS v3.2 Part 1 & 2 |
| Digital Twin Registry | Provider-hosted registry implementing the AAS Registry read profile (SSP-005) and Discovery read profile (SSP-002) | [sldt-digital-twin-registry](https://github.com/eclipse-tractusx/sldt-digital-twin-registry) |
| Submodel | The actual data for one aspect of a twin, served by the AAS Submodel API value profile (SSP-003), e.g. `/submodel/$value` | Any backend; usually behind the EDC data plane |
| Aspect model | RDF/Turtle schema in SAMM, identified by a URN such as `urn:samm:io.catenax.material_demand:1.0.0#MaterialDemand` | CX-0003; published in the Semantic Hub |
| Industry Core | Shared twin archetypes (part type, part instance) reused by many use cases | CX-0126, CX-0127, CX-0151 |

### How the DTR and submodels sit behind EDC

The DTR is itself an EDC asset. Its catalog entry must carry `dct:type` = `https://w3id.org/catenax/taxonomy#DigitalTwinRegistry` and a `cx-common:version` of at least `3.0`, and it should use policies that do not depend on use-case framework credentials. A single submodel exposed as its own dataset uses `dct:type` `...#Submodel`.

Each submodel descriptor in the DTR tells a consumer how to reach the data through EDC:

- `semanticId`: the aspect model URN.
- `subprotocol`: `DSP`.
- `subprotocolBody`: `id=<dataset id in the provider catalog>;dspEndpoint=<provider control plane URL>`.
- `href`: the provider data plane URL plus the AAS path.

So a consumer negotiates once for the DTR, reads twins, and then negotiates per submodel dataset named in `subprotocolBody`.

### Finding the right provider

Nobody can query every connector, so Catena-X layers discovery services (CX-0053):

1. **BPN Discovery**: maps an asset identifier (serial number, VIN, part number) to the BPN of who holds data about it.
2. **Discovery Finder**: tells you which BPN Discovery services exist for a given identifier type.
3. **Connector discovery**: maps a BPN to that company's connector DSP endpoint(s), using data registered at onboarding.
4. **BDRS**: maps the BPN to the DID that DCP needs.

The Item Relationship Service (IRS, CX-0005) builds on this to walk bills of material across companies tier by tier, which is how traceability use cases work.

### Why a backend team should care

- Most real integrations are **DTR + submodel server + EDC**, not EDC alone. Expect to build or operate a submodel service that maps internal data to SAMM JSON.
- Aspect model versions are part of the contract: per the Catena-X framework, a change of API version, asset version or aspect model version can require renegotiation unless the policy's `VersionChanges` constraint allows it ([framework agreement schema](https://w3id.org/catenax/2025/9/policy/framework-agreement-constraint-schema.json)).

## 13. End to end: a consumer fetches a supplier's submodel

Reading one piece of a supplier's data takes discovery, two DSP negotiations (first for the DTR, then for the submodel) and a final pull through the provider's data plane. Every DSP call carries a DCP token and triggers a credential check.

&#91;embedded content: Consumer A fetches a submodel from provider B · 10 calls\]

Solid arrows are requests, dashed arrows are the asynchronous DSP replies, and only the last call carries business data.

1. **Discover.** The app knows the supplier's BPN (or finds it via BPN Discovery from a part number). Connector discovery returns the supplier's DSP endpoint, and BDRS returns its DID.
2. **Get a token.** Connector A asks its STS for a self-issued ID token addressed to B's DID, embedding an access token scoped to the credentials it will show.
3. **Ask for the catalog.** A sends a DSP `CatalogRequestMessage` with that token, filtered for `dct:type` DigitalTwinRegistry.
4. **Verify A.** B resolves A's DID, checks the token signature, and queries A's Credential Service for a VP containing the MembershipCredential. Steps 2 and 4 repeat on every DSP call.
5. **Return the filtered catalog.** B evaluates each offer's access policy against A's verified claims and returns only the offers A may see.
6. **Negotiate for the DTR.** A requests the DTR offer; B checks the usage policy (here the FrameworkAgreement credential and UsagePurpose) in the `contract.negotiation` scope.
7. **Agree and start the transfer.** After the negotiation is FINALIZED, A requests an `HttpData-PULL` transfer and receives an EDR (endpoint + short-lived token) for B's data plane.
8. **Read the twin.** A calls B's data plane with the EDR; it proxies to the DTR. A looks up the shell by asset ID and reads the submodel descriptor, whose `subprotocolBody` names the submodel dataset id and B's DSP endpoint.
9. **Negotiate for the submodel.** Steps 3 to 7 repeat for that dataset, with its own usage policy.
10. **Pull the data.** A calls the descriptor's `href` (`.../submodel/$value`) with the new EDR. The data plane validates the token and proxies to B's submodel server, which returns JSON matching the SAMM aspect model.

In production, agreements and EDRs are cached and refreshed, so steps 6–7 and 9 happen once per agreement rather than once per read. Our services sit at both ends: the management API calls on A's side, and the DTR, submodel server and EDC configuration on B's side.

## 14. Component map: every building block, its project and what to know

Interoperability in Catena-X is defined by HTTP specifications, not by code, so every component below can be swapped for any implementation in any language that passes the same specs. The Tractus-X projects are reference implementations written in Java, Kotlin, C#, Python and TypeScript. Language only matters if you change a component's internals.

### Layer 0: the contracts (language-neutral)

These are what certification and interoperability are measured against. Learn these before any codebase.

| Contract | What it fixes | Where it is defined | How to check conformance |
| --- | --- | --- | --- |
| Dataspace Protocol 2025-1 | Catalog, negotiation, transfer between connectors | [DSP spec](https://eclipse-dataspace-protocol-base.github.io/DataspaceProtocol/2025-1/) | [DSP TCK](https://github.com/eclipse-dataspacetck/dsp-tck) |
| Decentralized Claims Protocol 1.0.1 | Identity tokens, presentations, issuance | [DCP spec](https://eclipse-dataspace-dcp.github.io/decentralized-claims-protocol/v1.0.1) | [DCP TCK](https://github.com/eclipse-dataspacetck/dcp-tck) |
| Data Plane Signaling | Control plane ↔ data plane | [eclipse-dataplane-signaling](https://github.com/eclipse-dataplane-signaling) | SDKs in Go, Rust, Java, .NET |
| AAS API v3.x | Digital Twin Registry and submodel APIs | IDTA AAS Part 2, profiled by [CX-0002](https://catenax-ev.github.io/docs/standards/CX-0002-DigitalTwinsInCatenaX) | Registry SSP-005, Discovery SSP-002, Submodel SSP-003 |
| SAMM aspect models | JSON payload schemas | [CX-0003](https://catenax-ev.github.io/docs/standards/CX-0003-SAMMSemanticAspectMetaModel), [sldt-semantic-models](https://github.com/eclipse-tractusx/sldt-semantic-models) | ESMF SDK / SAMM CLI validator |
| ODRL + Catena-X profile | Access and usage policies | [CX-0152](https://catenax-ev.github.io/docs/standards/CX-0152-PolicyConstrainsForDataExchange) | Published JSON Schemas |
| W3C DID, VC | Identity documents and credentials | W3C, profiled by CX-0049, CX-0050 | — |

### Layer 1: what every participant runs

| Building block | Needed when | Reference project | Stack | Interfaces | What developers must know |
| --- | --- | --- | --- | --- | --- |
| Connector control plane | Always | [tractusx-edc](https://github.com/eclipse-tractusx/tractusx-edc) on [eclipse-edc/Connector](https://github.com/eclipse-edc/Connector) | Java, Gradle; PostgreSQL, Vault | DSP (public), Management API v3/v4 (internal) | DSP states, ODRL policies, DCP scopes, EDRs |
| Connector data plane | Always | tractusx-edc data plane, or your own via Data Plane Signaling | Java; or any language via DPS SDKs | DPS from control plane, public data API | Token validation, transfer types, proxying to backends |
| Wallet + STS | Always | [tractusx-identityhub](https://github.com/eclipse-tractusx/tractusx-identityhub) (on [eclipse-edc/IdentityHub](https://github.com/eclipse-edc/IdentityHub)); commercial wallets | Java backend, TypeScript UI | DCP presentation + storage APIs, `did:web` | DIDs, VC/VP, key rotation, scopes |
| Digital Twin Registry | Twin-based use cases | [sldt-digital-twin-registry](https://github.com/eclipse-tractusx/sldt-digital-twin-registry) | Java | AAS Registry + Discovery API, behind EDC | Shell descriptors, `specificAssetIds`, `subprotocolBody` |
| Submodel service | Twin-based use cases | Any AAS-compliant server; `simple-data-backend` in the umbrella repo for tests | Any | AAS Submodel API (`/$value`) | SAMM JSON, versioned aspect URNs |
| Integration / orchestration | Optional, very common | [Industry Core Hub](https://github.com/eclipse-tractusx/industry-core-hub), [tractusx-sdk](https://github.com/eclipse-tractusx/tractusx-sdk), [tractusx-edc-dashboard](https://github.com/eclipse-tractusx/tractusx-edc-dashboard) | Python (FastAPI), TypeScript (React, Angular) | Calls Management API, DTR, discovery | Which Management API version; policy templates |
| Use-case applications | Per use case | [PURIS](https://github.com/eclipse-tractusx/puris), [IRS](https://github.com/eclipse-tractusx/item-relationship-service), [Trace-X](https://github.com/eclipse-tractusx/traceability-foss) | Java; Trace-X adds Angular | Use-case APIs over EDC | The use case's KIT and standards |

The Industry Core Hub is the most complete language-neutral example of the whole provider and consumer flow: a Python/FastAPI backend that registers twins in the DTR, creates EDC assets and policies, and consumes partner data via discovery, built on the Tractus-X SDK ([README](https://github.com/eclipse-tractusx/industry-core-hub)).

### Layer 2: shared core services (run by the operator; we mostly call them)

| Service | Project | Stack | Who calls it | Note |
| --- | --- | --- | --- | --- |
| Portal, onboarding, connector registration, EDC discovery | [portal-backend](https://github.com/eclipse-tractusx/portal-backend), [portal-frontend](https://github.com/eclipse-tractusx/portal-frontend), portal-iam | C# (.NET), TypeScript, Keycloak | People; apps for EDC discovery | EDC discovery is served by the portal backend |
| Credential issuer (current production path) | [ssi-credential-issuer](https://github.com/eclipse-tractusx/ssi-credential-issuer) | C# (.NET 9) | Portal; wallets | Talks to the DIV (formerly DIM) wallet via SAP's SSI DIM Middle Layer |
| Credential issuer (DCP-native) | IssuerService in tractusx-identityhub | Java | Holders' IdentityHubs via DCP issuance | Not integrated with the portal backend as of 26.06; 26.09 still relies on a commercial wallet bridge |
| SSI Authority & Schema Registry | [ssi-authority-schema-registry](https://github.com/eclipse-tractusx/ssi-authority-schema-registry) | C# (.NET 8) | Issuers, verifiers | Repository archived January 2026; kept here because older docs still reference it |
| BDRS | [bpn-did-resolution-service](https://github.com/eclipse-tractusx/bpn-did-resolution-service) | Java (EDC runtime) | Every connector | BPN ↔ DID directory |
| Business partner data (BPDM) | [bpdm](https://github.com/eclipse-tractusx/bpdm) | Kotlin | Partner-data apps | Golden records; BPN issuing |
| Discovery Finder, BPN Discovery | [sldt-discovery-finder](https://github.com/eclipse-tractusx/sldt-discovery-finder), [sldt-bpn-discovery](https://github.com/eclipse-tractusx/sldt-bpn-discovery) | Java (Spring Boot, Maven) | Consumer apps | Asset ID → BPN lookups (CX-0053) |
| Semantic Hub | [sldt-semantic-hub](https://github.com/eclipse-tractusx/sldt-semantic-hub) | Java | Model tooling | Tagged a deprecation candidate; the models repo is the real source |
| Self-Description Factory | [sd-factory](https://github.com/eclipse-tractusx/sd-factory) | Java | Portal at onboarding | Builds Gaia-X self-descriptions |

### Layer 3: developer tooling

| Tool | Project | Stack | Use it for |
| --- | --- | --- | --- |
| Full local dataspace | [tractus-x-umbrella](https://github.com/eclipse-tractusx/tractus-x-umbrella) | Helm | Portal, connectors, wallet, DTR, discovery on one Kubernetes cluster |
| Guided labs | [tutorial-resources](https://github.com/eclipse-tractusx/tutorial-resources) | Terraform (HCL) | Workshops; its MXD lab still pins connector chart 0.9.0, so use the umbrella (section 16) for current versions |
| Dataspace test framework | [tractusx-testlab](https://github.com/eclipse-tractusx/tractusx-testlab) | Python (first alpha in 26.09) | Automated end-to-end test cases on the SDK |
| Onboarding checks | [tractusx-sdk-services](https://github.com/eclipse-tractusx/tractusx-sdk-services) | Python | Test orchestrator and digital-twin pull service |
| API catalogue | [api-hub](https://github.com/eclipse-tractusx/api-hub) | OpenAPI | Generating clients in any language |
| Semantic tooling | [eclipse-esmf](https://github.com/eclipse-esmf) | Java SDK + SAMM CLI, Python and JS loaders, Aspect Model Editor | Validating and generating code from aspect models |
| Upstream demo | [EDC MVD](https://github.com/eclipse-dataspace-hub/MinimumViableDataspace) | Java | Smallest working DSP + DCP setup |

### Retired or replaced — do not start new work here

- [policy-hub](https://github.com/eclipse-tractusx/policy-hub) (C#): archived December 2025.
- [digital-product-pass](https://github.com/eclipse-tractusx/digital-product-pass) (Java + JS): archived October 2025; the README points to the Industry Core Hub instead.
- [ssi-authority-schema-registry](https://github.com/eclipse-tractusx/ssi-authority-schema-registry) (C#): archived January 2026.
- managed-identity-wallet: replaced by IdentityHub (section 18).
- Federated catalog extension in tractusx-edc: removed in 0.13.0.

Repository languages are from the [Tractus-X GitHub organisation](https://github.com/orgs/eclipse-tractusx/repositories) and each project's README; the tractusx-identityhub repo shows TypeScript as its top language only because of its UI folder.

### Upstream: what each Tractus-X project is built on

Only the connector, the wallet and a few tools sit on Eclipse upstream code; most other Tractus-X projects are Tractus-X's own code implementing open specs. For upstream-based projects, read upstream docs and code at the version Tractus-X pins (tractusx-edc 0.13.0 = EDC 0.17.0), not upstream main.

| Tractus-X project | Built on (Eclipse upstream) | What Tractus-X adds | Go upstream for |
| --- | --- | --- | --- |
| [tractusx-edc](https://github.com/eclipse-tractusx/tractusx-edc) | [eclipse-edc/Connector](https://github.com/eclipse-edc/Connector), EDC 0.17.0 in release 0.13.0 | Catena-X extensions (DCP defaults, BDRS client, cx-policy, EDR API, DSP 0.8), Helm charts, images | State machines, policy engine, DSP and DCP implementation, Management API |
| [tractusx-identityhub](https://github.com/eclipse-tractusx/tractusx-identityhub) | [eclipse-edc/IdentityHub](https://github.com/eclipse-edc/IdentityHub), EDC 0.18.0 on main | Catena-X setup, Helm charts, images, the IssuerService distribution | Wallet internals, DCP presentation and issuance |
| [bpn-did-resolution-service](https://github.com/eclipse-tractusx/bpn-did-resolution-service) | EDC runtime libraries (EDC 0.16.1) | The BPN ↔ DID directory itself | Extension model and configuration |
| [tractusx-edc-dashboard](https://github.com/eclipse-tractusx/tractusx-edc-dashboard) (the Catena-X Policy Builder) | [eclipse-edc/DataDashboard](https://github.com/eclipse-edc/DataDashboard) as a Git submodule, plus an in-memory EDC runtime for validation | Catena-X policy builder UI | Dashboard components; upstream calls it demo-grade, not production code |
| [item-relationship-service](https://github.com/eclipse-tractusx/item-relationship-service) | EDC libraries in its EDC client, pinned at EDC 0.6.0 | Bill-of-material traversal across companies | Nothing current: the pin is old, so don't copy its EDC patterns |
| [sldt-semantic-hub](https://github.com/eclipse-tractusx/sldt-semantic-hub), [sldt-semantic-models](https://github.com/eclipse-tractusx/sldt-semantic-models) | [eclipse-esmf](https://github.com/eclipse-esmf): the [SAMM specification](https://github.com/eclipse-esmf/esmf-semantic-aspect-meta-model) and [ESMF SDK](https://github.com/eclipse-esmf/esmf-sdk) | Catena-X aspect models and the hub that serves them | Modelling rules, validation, code generation |
| tractusx-edc test suites | [dsp-tck](https://github.com/eclipse-dataspacetck/dsp-tck), [dcp-tck](https://github.com/eclipse-dataspacetck/dcp-tck) | Runs both TCKs in CI | What protocol conformance means |

Tractus-X's own code, with no upstream base: DTR (implements the IDTA AAS API), portal, ssi-credential-issuer, BPDM, the discovery services, PURIS, Trace-X, tractusx-sdk and the Industry Core Hub. For these, the Tractus-X repo is the source of truth.

Upstream specs and repos to follow:

- Specs: [Dataspace Protocol](https://github.com/eclipse-dataspace-protocol-base/DataspaceProtocol), [Decentralized Claims Protocol](https://github.com/eclipse-dataspace-dcp/decentralized-claims-protocol) and [Data Plane Signaling](https://github.com/eclipse-dataplane-signaling/dataplane-signaling), all developed in the open on GitHub.
- Next generation: [eclipse-edc/Virtual-Connector](https://github.com/eclipse-edc/Virtual-Connector) holds the virtualized control plane behind EDC-V and Management API v5 (section 8).
- Moved: the standalone [FederatedCatalog](https://github.com/eclipse-edc/FederatedCatalog) repo is archived and its modules now live in the Connector repo.
- Where a fix belongs: a bug in a state machine, the policy engine or the protocol code is fixed upstream and reaches us with the next EDC version bump in tractusx-edc. Catena-X policies, BDRS lookups, EDR handling and Helm charts are fixed in the Tractus-X repo.

## 15. Working in any language: what each stack owns

Most Catena-X work is integration, and integration happens over four HTTP boundaries: the connector's Management API, the EDR/data plane endpoint, the DTR and submodel AAS APIs, and the discovery services. Any language that speaks HTTP and JSON can own that work. You need the JVM only to change what happens *inside* EDC, IdentityHub, DTR or BDRS.

### Pick the boundary before the language

| You want to… | Integrate at | Language constraint |
| --- | --- | --- |
| Offer or consume data from our systems | Our connector's Management API + EDR/proxy endpoint | None |
| Publish digital twins | DTR registry API + a submodel server | None |
| Move data with a protocol EDC doesn't ship | Implement a data plane via Data Plane Signaling | None (SDKs: Go, Rust, Java, .NET) |
| Add a Catena-X policy constraint, custom DCP scope mapping or store | EDC / IdentityHub extension | JVM (Java/Kotlin) |
| Run a different connector product | DSP + DCP specs, verified with the TCKs | None, but expensive |

### Tracks

| Track | Typically owns | Start with | Can treat as a black box |
| --- | --- | --- | --- |
| JVM (Java/Kotlin) | EDC extensions (policy functions, transfer types), IdentityHub extensions, contributions to DTR, BDRS, IRS, PURIS; BPDM is Kotlin | Sections 8–10, EDC extension model, tractusx-edc `edc-extensions/` | Portal, issuer, UIs |
| Python | Provider/consumer orchestration, data pipelines into submodels, automated dataspace tests | [tractusx-sdk](https://pypi.org/project/tractusx-sdk/), Industry Core Hub backend, tractusx-testlab, ESMF Python loader | EDC internals; drive it through the Management API |
| TypeScript/JS | Operator and business UIs, policy builders, aspect-model-driven forms | portal-frontend, IC-Hub frontend, tractusx-edc-dashboard, ESMF JS loader | Everything server-side |
| C#/.NET | Portal backend, credential issuer; .NET data planes | portal-backend, ssi-credential-issuer, Data Plane SDK for .NET | Connector internals |
| Go / Rust | High-throughput or streaming data planes; infra tooling | Data Plane Signaling SDKs for Go and Rust | Control plane and wallet |

### Rules of thumb for every track

1. **Never call a partner's connector from application code.** Apps call *our* connector (Management API) and *our* data plane or EDR proxy; the connector does DSP and DCP.
2. **Generate clients from OpenAPI.** Use the specs in the [api-hub](https://github.com/eclipse-tractusx/api-hub) or the EDC Management API spec instead of hand-writing JSON-LD.
3. **Pin the Management API version.** With tractusx-edc 0.13.x, write new clients against v4 for upstream resources (assets, policies, contract definitions, catalog, negotiations, transfers), and use v3 only for Tractus-X endpoints with no v4 yet (EDRs, BPN groups, connector discovery). The two coexist for a while, and v4 validates payloads with JSON Schema, which makes plain-JSON clients in any language much easier ([EDC blog](https://eclipse-edc.github.io/blog/2026/02/19/management-api-upcoming-versions/)).
4. **Treat aspect models as the schema of record.** Generate types from SAMM (ESMF SDK or loaders) rather than copying example JSON.
5. **Prove interoperability with tests, not language parity.** Use the umbrella chart for end-to-end runs and the DSP/DCP TCKs if you ever build protocol code.

The JVM-specific material in sections 8–10 is for the people extending the connector or wallet; everyone else should read those sections for concepts and skip the extension details.

## 16. Hands-on: your first week

By the end of week one, every new engineer should have published a passport-style asset from one connector and pulled it from another, using only the Management API. Run it on the umbrella chart's IdentityHub scenario. The calls below follow the [tractusx-edc Management API walkthrough](https://github.com/eclipse-tractusx/tractusx-edc/blob/main/docs/usage/management-api-walkthrough/README.md), which uses v3; for new product code, follow the version advice in section 15.

### The week

| Day | Goal | What to do |
| --- | --- | --- |
| 1 | Vocabulary | Sections 1–7 of this doc, then Phase 1 of the reading list |
| 2 | A dataspace on your laptop | Install [tractus-x-umbrella](https://github.com/eclipse-tractusx/tractus-x-umbrella) with its [decentralized IdentityHub profile](https://github.com/eclipse-tractusx/tractus-x-umbrella/blob/main/docs/user/common/guides/data-exchange-identityhub.md): a provider and a consumer connector, each with an IdentityHub, plus IssuerService and BDRS |
| 3 | Provide | Steps 1–3 below, on the provider |
| 4 | Consume | Steps 4–6 below on the consumer, then the DTR extension (steps 7–10); watch the negotiation and transfer rows change in PostgreSQL |
| 5 | Break it | Remove a DCP scope, change the usage purpose, let a token expire; match each failure to the pitfalls in section 18 |

The umbrella pins its own component versions (its main branch was at release 26.03 when this was written), so compare them with section 18 before drawing conclusions about production.

### JSON-LD in five minutes

Every Management API and DSP payload is JSON-LD: ordinary JSON in which every key stands for a full IRI.

- @context maps short names to IRIs. With "@vocab": "https://w3id.org/edc/v0.0.1/ns/", a bare key such as baseUrl means https://w3id.org/edc/v0.0.1/ns/baseUrl.
- Prefixes declared in the context shorten other vocabularies: dct: (http://purl.org/dc/terms/), odrl: (http://www.w3.org/ns/odrl/2/), dcat: (https://www.w3.org/ns/dcat/), cx-taxo: (https://w3id.org/catenax/taxonomy#), tx-auth: (https://w3id.org/tractusx/auth/).
- @id is the object's identifier and @type its class, e.g. Asset, PolicyDefinition or ContractRequest.
- EDC expands every request to full IRIs and compacts its responses again. A response may therefore use different prefixes from the ones you sent, and a one-item list can come back as a single object. Read responses by IRI or through a JSON-LD library (Titanium in Java, PyLD in Python, jsonld.js in JavaScript), never by string matching.
- Catena-X policies need the policy context https://w3id.org/catenax/2025/9/policy/context.jsonld. Without it, FrameworkAgreement and UsagePurpose do not resolve and policy validation rejects the request.
- Paste any payload into the [JSON-LD Playground](https://json-ld.org/playground/) to see its expanded form.

### Your first exchange, request by request

Every call goes to a connector's Management API with an X-Api-Key header. Provider and consumer URLs, DIDs and keys come from the umbrella guide.

These calls use v3, like the Tractus-X walkthrough. In v4 the paths start with /v4, and @context must be an array of URLs that includes https://w3id.org/edc/connector/management/v2. Inline prefix objects are not allowed there, so write terms such as dct:type as full IRIs (http://purl.org/dc/terms/type), and every body is checked against the [v4 JSON Schemas](https://github.com/eclipse-edc/Connector/tree/v0.17.0/extensions/common/api/management-api-schema-validator/src/main/resources/schema/management/v4).

1. Provider: register the passport submodel as an asset. The dataAddress points at the backend the data plane will proxy to.

```http
POST {{PROVIDER_MGMT}}/v3/assets
X-Api-Key: {{PROVIDER_KEY}}
Content-Type: application/json

{
  "@context": { "@vocab": "https://w3id.org/edc/v0.0.1/ns/", "dct": "http://purl.org/dc/terms/" },
  "@type": "Asset",
  "@id": "battery-pass-0001",
  "properties": { "dct:type": { "@id": "https://w3id.org/catenax/taxonomy#Submodel" } },
  "dataAddress": {
    "@type": "DataAddress",
    "type": "HttpData",
    "baseUrl": "https://submodels.internal/battery-pass/0001",
    "proxyPath": "true",
    "proxyQueryParams": "true"
  }
}
```

2. Provider: create two policies. dpp-access has action access and the constraint Membership eq active; dpp-usage below is the usage policy every passport offer needs.

```http
POST {{PROVIDER_MGMT}}/v3/policydefinitions
X-Api-Key: {{PROVIDER_KEY}}
Content-Type: application/json

{
  "@context": [
    "https://w3id.org/dspace/2025/1/odrl-profile.jsonld",
    "https://w3id.org/catenax/2025/9/policy/context.jsonld",
    { "@vocab": "https://w3id.org/edc/v0.0.1/ns/" }
  ],
  "@type": "PolicyDefinition",
  "@id": "dpp-usage",
  "policy": {
    "@type": "Set",
    "permission": [{
      "action": "use",
      "constraint": [{ "and": [
        { "leftOperand": "FrameworkAgreement", "operator": "eq", "rightOperand": "DataExchangeGovernance:1.0" },
        { "leftOperand": "UsagePurpose", "operator": "isAnyOf", "rightOperand": "cx.circular.dpp:1" }
      ]}]
    }]
  }
}
```

3. Provider: bind the asset to both policies with a contract definition.

```http
POST {{PROVIDER_MGMT}}/v3/contractdefinitions
X-Api-Key: {{PROVIDER_KEY}}
Content-Type: application/json

{
  "@context": { "@vocab": "https://w3id.org/edc/v0.0.1/ns/" },
  "@type": "ContractDefinition",
  "@id": "battery-pass-0001-offer",
  "accessPolicyId": "dpp-access",
  "contractPolicyId": "dpp-usage",
  "assetsSelector": {
    "@type": "CriterionDto",
    "operandLeft": "https://w3id.org/edc/v0.0.1/ns/id",
    "operator": "=",
    "operandRight": "battery-pass-0001"
  }
}
```

4. Consumer: request the provider's catalog. Over DSP 2025-1, counterPartyId is the provider's DID; over DSP 0.8 it is its BPN.

```http
POST {{CONSUMER_MGMT}}/v3/catalog/request
X-Api-Key: {{CONSUMER_KEY}}
Content-Type: application/json

{
  "@context": { "@vocab": "https://w3id.org/edc/v0.0.1/ns/" },
  "@type": "CatalogRequest",
  "counterPartyId": "{{PROVIDER_DID}}",
  "counterPartyAddress": "https://provider.example/api/v1/dsp/2025-1",
  "protocol": "dataspace-protocol-http:2025-1"
}
```

The dataset for battery-pass-0001 carries an offer under odrl:hasPolicy. Copy the offer's @id and body exactly as returned.

5. Consumer: negotiate and start the transfer in one call through the Tractus-X EDR API.

```http
POST {{CONSUMER_MGMT}}/v3/edrs
X-Api-Key: {{CONSUMER_KEY}}
Content-Type: application/json

{
  "@context": [
    "https://w3id.org/catenax/2025/9/policy/context.jsonld",
    "https://w3id.org/dspace/2025/1/odrl-profile.jsonld",
    { "@vocab": "https://w3id.org/edc/v0.0.1/ns/" }
  ],
  "@type": "ContractRequest",
  "counterPartyAddress": "https://provider.example/api/v1/dsp/2025-1",
  "protocol": "dataspace-protocol-http:2025-1",
  "policy": {
    "@id": "{{OFFER_ID}}",
    "@type": "Offer",
    "assigner": "{{PROVIDER_DID}}",
    "target": "battery-pass-0001",
    "permission": [ "…copied unchanged from the catalog…" ]
  },
  "callbackAddresses": []
}
```

6. Consumer: find the transfer, fetch its EDR and pull the data.

```http
POST {{CONSUMER_MGMT}}/v3/edrs/request                        (QuerySpec filtered on assetId; returns the transferProcessId)
GET  {{CONSUMER_MGMT}}/v3/edrs/{transferProcessId}/dataaddress?auto_refresh=true
GET  {endpoint from the EDR}
Authorization: {authorization from the EDR}
```

The EDR holds the provider's data plane endpoint, a short-lived authorization token (tx-auth:expiresIn, 300 seconds in the walkthrough) and a refresh token; auto\_refresh=true renews the token for you.

Check yourself at the end of the week:

- Which steps caused DCP traffic? Steps 4 and 5: every DSP message the consumer's connector sent carried a self-issued token, and the provider queried the consumer's IdentityHub for credentials.
- Why must the offer go back unchanged? The provider compares it with its own offer, and any difference ends the negotiation in TERMINATED.
- What would the asynchronous PCF flow need that this exercise lacked? An offer on the consumer's side too, because the provider answers with a PUT to the consumer's exchange asset (section 4).

### Day 4 extension: find the passport through the Digital Twin Registry

Real passports are found through the provider's DTR, not by asset id (section 12). The umbrella's provider ships a DTR and a simple submodel backend, so add these steps once steps 1–6 work. The request shapes follow [CX-0002](https://catenax-ev.github.io/docs/standards/CX-0002-DigitalTwinsInCatenaX) and the [Digital Twin KIT](https://eclipse-tractusx.github.io/docs-kits/kits/digital-twin-kit/software-development-view).

7. Provider: register a twin with a submodel descriptor, calling the DTR on its internal URL rather than through EDC.

```http
POST {{PROVIDER_DTR}}/api/v3/shell-descriptors
Content-Type: application/json

{
  "id": "urn:uuid:{{SHELL_ID}}",
  "idShort": "battery-0001",
  "specificAssetIds": [{
    "name": "manufacturerPartId",
    "value": "BAT-0001",
    "externalSubjectId": { "type": "ExternalReference",
      "keys": [{ "type": "GlobalReference", "value": "PUBLIC_READABLE" }] }
  }],
  "submodelDescriptors": [{
    "id": "urn:uuid:{{SUBMODEL_ID}}",
    "semanticId": { "type": "ExternalReference",
      "keys": [{ "type": "GlobalReference",
        "value": "urn:samm:io.admin-shell.idta.batterypass.digital_nameplate:1.0.0#BatteryNameplate" }] },
    "endpoints": [{
      "interface": "SUBMODEL-3.0",
      "protocolInformation": {
        "href": "https://provider.example/api/public",
        "endpointProtocol": "HTTP",
        "endpointProtocolVersion": ["1.1"],
        "subprotocol": "DSP",
        "subprotocolBody": "id=battery-pass-0001;dspEndpoint=https://provider.example/api/v1/dsp",
        "subprotocolBodyEncoding": "plain"
      }
    }]
  }]
}
```

The DTR is closed by default: a specificAssetId is visible only to BPNs listed in its externalSubjectId, and PUBLIC\_READABLE is allowed only for manufacturerPartId and assetLifecyclePhase ([DTR access rules](https://github.com/eclipse-tractusx/sldt-digital-twin-registry/blob/main/docs/architecture/6-crosscutting-concepts.md)). A consumer whose lookup comes back empty is usually missing from that list.

8. Provider: offer the DTR itself as an asset whose baseUrl is the DTR's /api/v3, with dct:type https://w3id.org/catenax/taxonomy#DigitalTwinRegistry, a cx-common:version of 3.0 and policies that need no use-case credential (section 12). Skip this if the provider's catalog already lists one.
9. Consumer: request the catalog filtered on that type, negotiate for the DTR offer as in step 5, and look up the twin through the provider's data plane with the DTR's EDR.

```http
"filterExpression": [{ "operandLeft": "'http://purl.org/dc/terms/type'.'@id'", "operator": "=",
                       "operandRight": "https://w3id.org/catenax/taxonomy#DigitalTwinRegistry" }]

GET {endpoint from the DTR EDR}/lookup/shells?assetIds={base64url of {"name":"manufacturerPartId","value":"BAT-0001"}}
GET {endpoint from the DTR EDR}/shell-descriptors/{base64url of the shell id}
Authorization: {authorization from the DTR EDR}
```

10. Consumer: read subprotocolBody from the submodel descriptor, negotiate for dataset battery-pass-0001 at that dspEndpoint (steps 4–5), then call `href + /submodel/$value` with the new EDR. The data plane forwards that path to the asset's baseUrl, so the backend must answer at `…/battery-pass/0001/submodel/$value`.

### Knowing when things finish

Creating a negotiation, a transfer or an EDR returns only an id; the work then runs asynchronously over DSP. There are three ways for a backend to learn the outcome.

| Approach | How | Use when |
| --- | --- | --- |
| Poll | GET /v3/contractnegotiations/{id} until FINALIZED or TERMINATED; GET /v3/transferprocesses/{id} until STARTED | Scripts, tests, low volume |
| Callbacks | Add callbackAddresses to the request; our own connector POSTs an event to our backend on each matching state change | Product backends |
| EDR API | POST /v3/edrs negotiates and transfers in one call; subscribe only to transfer.process.started, then read the EDR | Pull transfers, which is most Catena-X traffic |

A callback entry looks like this. Events are hierarchical, so contract.negotiation subscribes to every negotiation state change.

```json
"callbackAddresses": [{
  "uri": "https://our-backend.internal/edc-events",
  "events": ["contract.negotiation", "transfer.process.started"],
  "transactional": false,
  "authKey": "Authorization",
  "authCodeId": "edc-callback-secret"
}]
```

authCodeId names a secret in our Vault, which the connector sends in the authKey header ([negotiation walkthrough](https://github.com/eclipse-tractusx/tractusx-edc/blob/main/docs/usage/management-api-walkthrough/05_contractnegotiations.md)). Design the handler for duplicates and gaps: make it idempotent, and keep a slow reconciliation poll for events that never arrive.

## 17. Running connectors for customers

For Connector and Dataspace as a Service, operating connectors is the product. Each customer deployment needs the controls below before a partner can rely on it. The endpoint defaults come from the [tractusx-connector Helm chart](https://github.com/eclipse-tractusx/tractusx-edc/blob/main/charts/tractusx-connector/values.yaml).

### Onboarding a customer, step by step

Catena-X onboarding is invitation-based: a company cannot register itself without an Onboarding Service Provider (OSP) or the CSP-B ([CX-0006](https://catenax-ev.github.io/docs/standards/CX-0006-RegistrationAndInitialOnboarding)). Steps 1–4 belong to the operator; steps 5–8 are ours.

1. **Registration request.** An OSP, or the CSP-B directly, collects the company's name and full address and sends a registration request to the CSP-B.
2. **Data collection.** The CSP-B collects the data needed for a BPNL, whether the company brings its own wallet or takes one from the CSP-B, and which roles it wants. Roles that need certification require a certificate from a Conformity Assessment Body.
3. **Validation and identity proofing.** The CSP-B checks that the company is not already onboarded, verifies its certificates, creates the BPNL if needed, and proves the company's identity under the [identity-proofing rulebook](https://catenax-ev.github.io/docs/rulebooks/CX-NFR-IdP).
4. **Wallet and credentials.** The company gets a wallet that meets CX-0149. The CSP-B issues a BPN credential, a framework agreement credential and the MembershipCredential; issuing the MembershipCredential completes onboarding.
5. **Subscription.** The customer subscribes to our offer in the portal's service marketplace. We publish that offer once, through the portal's [service release process](https://github.com/eclipse-tractusx/portal-assets/blob/main/docs/user/05.%20Service%28s%29/02.%20Service%20Release%20Process/index.md), and the operator approves it.
6. **Activation.** We [activate the subscription](https://github.com/eclipse-tractusx/portal-assets/blob/main/docs/user/05.%20Service%28s%29/03.%20Service%20Subscription/04.%20Service%20Subscription%20Activation%20%28Provider%29.md): install the customer's tenant, receive the technical user the portal creates in the customer's name, and store the service URL.
7. **Connector registration.** We register the connector in the customer's name with the portal's Connector-as-a-service option, by UI or API; it needs our company to hold the SERVICE\_PROVIDER or APP\_PROVIDER role. Registration triggers the connector's self-description, and the connector stays PENDING for about a minute ([portal guide](https://github.com/eclipse-tractusx/portal-assets/blob/main/docs/user/02.%20Technical%20Integration/01.%20Connector%20Registration/02.%20Connector%20Registration.md)).
8. **Configuration and smoke test.** Set the customer's DID and BPNL (participant.id, participant.bpnl), its wallet or STS, the trusted issuers and the BDRS URL. Then request a partner's catalog and run one transfer.

The portal guides above are the Tractus-X reference docs, last updated in September 2025. The production operator's portal may differ, so confirm each step with the operator before automating it.

### What to expose where

| Component | Endpoint (Helm default port, path) | Exposure |
| --- | --- | --- |
| Control plane | default: 8080, /api (health checks under /api/check) | Cluster only |
| Control plane | management: 8081, /management | Internal only; X-Api-Key, or delegated auth through a JWKS URL |
| Control plane | control: 8083, /control | Cluster only |
| Control plane | protocol (DSP): 8084, /api/v1/dsp | Internet: partners' connectors call it |
| Data plane | public: 8081, /api/public | Internet: consumers pull data here |
| Data plane | consumer proxy: 8186 | Internal only, with its own API key |
| Both planes | metrics: 9090, /metrics | Monitoring network only |
| IdentityHub | DCP API and DID endpoint | Internet; the Identity (admin) API stays internal (section 10) |

### Checklist per customer deployment

- **Secrets.** The chart ships the management key "password". Replace it with one key per customer, kept in Vault, and prefer delegated auth through a JWKS URL for our own product backends.
- **Health.** The chart wires Kubernetes probes to /api/check/liveness and /api/check/readiness. Also probe what partners actually hit from outside the cluster: the DSP endpoint and the DID document over HTTPS.
- **Logs and telemetry.** The chart's default log level is DEBUG; lower it in production. Tractus-X chose structured JSON logs and OpenTelemetry for traces and domain events, with metrics computed in an OpenTelemetry collector rather than in the connector ([decision record](https://github.com/eclipse-tractusx/tractusx-edc/blob/main/docs/development/decision-records/2025-07-09-logging_and_metrics_strategy/README.md)).
- The umbrella chart ships an example telemetry stack of OpenTelemetry collector, Jaeger, Prometheus, Loki and Grafana ([observability guide](https://github.com/eclipse-tractusx/tractus-x-umbrella/blob/main/docs/user/common/guides/observability/observability.md)).
- **What to alert on.** Negotiations stuck before FINALIZED, transfers ending in TERMINATED (read errorDetail), 401s at the DSP endpoint, EDR refresh failures, and customer credentials that expire within 30 days.
- **Identity settings.** Since tractusx-edc 0.13.0, participant.id holds the connector's DID and participant.bpnl its BPNL, and edc.participant.context.id must never change once set ([0.13 migration guide](https://github.com/eclipse-tractusx/tractusx-edc/blob/main/docs/migration/2026_06-Version_0.12.x_0.13.x.md)).
- **Keys.** Use one Vault path per customer. IdentityHub can sign through Vault's Transit engine so private keys never leave Vault; schedule key rotation (section 10).
- **Upgrades.** Tractus-X releases quarterly. Read every migration guide you cross and upgrade a canary customer first. 0.13.0, for example, replaced the Bitnami PostgreSQL chart with PostgreSQL 18 and offers no automatic upgrade path.
- **Partner tickets.** When a partner reports that our connector is down, check in this order: the public DSP endpoint, the DID document, credential validity, their issuer in our trusted list, and whether both sides speak the same DSP version.

## 18. Versions, renames and pitfalls

The ecosystem is mid-migration on three axes at once: protocol versions (DSP 0.8 → 2025-1, DCP 0.8 → 1.0), identity keys (BPN → DID), and component generations (central wallet → IdentityHub, DPF → Data Plane Signaling). Always ask "which version does the other side speak?" before debugging anything else.

### Names you will see, and what they mean today

| Old name in code/config | Current name | Note |
| --- | --- | --- |
| IATP (Identity and Trust Protocol) | DCP (Decentralized Claims Protocol) | Renamed when it moved from Tractus-X to an Eclipse spec project; tractusx-edc 0.13.0 renamed its config ([release](https://github.com/eclipse-tractusx/tractusx-edc/releases/tag/0.13.0)) |
| `.../iatp/...` wallet endpoint | `.../dcp/...` | Cofinity-X stopped accepting the iatp URL for new wallet service accounts in June 2026 ([notice](https://userguide.cofinity-x.com/en/articles/710191-june-2026-iatp-wallet-endpoint-deprecation)) |
| DIM (wallet) | DIV | Renamed in tractusx-edc 0.13.0 |
| Managed Identity Wallet (MIW) | tractusx-identityhub | Tractus-X restarted wallet work on upstream IdentityHub ([mailing list](https://www.eclipse.org/lists/tractusx-dev/msg00640.html)) |
| IDS / IDS protocol | DSP | DSP grew out of IDSA work and is now an Eclipse spec |
| Summary Credential | Individual credentials (Membership, BPN, DataExchangeGovernance) | Very old Catena-X releases used one summary VC |
| Management API v2 / v3 | v4 (then v5) | v3 deprecated in EDC 0.17.0 (tractusx-edc 0.13.0); v5 in beta |
| participant.id = BPNL (Helm value) | participant.id = DID; participant.bpnl = BPNL | Changed in tractusx-edc 0.13.0; the DID used to sit in iatp.id (section 17) |

### Version compatibility to keep in mind

- **Jupiter-era connectors** speak DSP 0.8 and DCP 0.8. tractusx-edc 0.13.0 had to disable its compatibility tests because the then-current IdentityHub could not handle DCP 0.8 ([0.13.0-rc3](https://github.com/eclipse-tractusx/tractusx-edc/releases/tag/0.13.0-rc3)). Mixed fleets are normal; verify both sides.
- **Tractus-X 26.09** (16 September 2026) is the first train that ships tractusx-edc 0.13.0, together with DTR 0.13.0 and IdentityHub 0.4.1 (holder wallet and IssuerService). IdentityHub is released as a component, but the release notes still say the release "does not include an Identity Wallet": it relies on a temporary bridge to a commercial wallet through SAP's open-source SSI DIM Middle Layer, without which participants cannot use the release fully. IRS was tested only against connector 0.12.2 with the Jupiter policy profile ([26.09 changelog](https://eclipse-tractusx.github.io/blog-changelog/release-26-09)). Never assume components in one train were tested together beyond what the known-issues list says.
- **BPN → DID**: Tractus-X 25.06 moved the SSI authority from BPN to DID, removing the BDRS lookup step for issuers ([25.06 plan](https://projects.eclipse.org/projects/automotive.tractusx/releases/25.06/plan)). Connectors still need BDRS while policies and partner APIs use BPNs.
- **EDC 1.0** release candidates (up to RC4 by 10 October 2026) exist upstream but are not yet in a Tractus-X distribution.

### Pitfalls we should expect

1. **Empty catalog** → the access policy failed. Check DCP default scopes, trusted issuer DIDs, and that our DID document is reachable over HTTPS from outside.
2. **Negotiation TERMINATED** → usage policy mismatch. Compare the requested offer byte-for-byte with the provider's catalog entry and check the FrameworkAgreement credential.
3. **401 at the provider DSP endpoint** → self-issued token problems: wrong `aud`, clock skew, an unpublished or rotated key.
4. **Data plane 403** → EDR token expired; use the EDR refresh endpoint instead of re-negotiating.
5. **Works locally, fails in Catena-X** → local tests used a mock issuer; production trusts only the CSP-B issuer DID.
6. **Vault surprises** → HashiCorp Vault 1.18.1 is known to break EDC path handling (README).
7. **Helm upgrade breaks config** → renamed keys (`iatp`→`dcp`, `dim`→`div`); read each migration guide you cross.

## 19. Glossary

| Term | Meaning |
| --- | --- |
| AAS | Asset Administration Shell: IDTA standard for digital twins and their APIs |
| Access policy | Policy deciding who may see an offer in the catalog |
| ASR | SSI Authority & Schema Registry: which authority may issue which credential type (repository archived January 2026) |
| BDRS | BPN-DID Resolution Service: central directory mapping BPNs to DIDs |
| BPN / BPNL / BPNS / BPNA | Business Partner Number for a legal entity, site or address |
| BPDM | Business Partner Data Management: the golden-record services that issue BPNs |
| CSP-A / CSP-B | Core Service Providers; CSP-B is the single operator of identity and discovery core services |
| Contract definition | EDC entity binding assets to an access policy and a contract policy |
| Control plane | EDC component that speaks DSP, evaluates policies, persists negotiations and transfers |
| Credential Service | Wallet endpoint a verifier calls to get a VP; advertised in the DID document |
| Data plane | Component that moves the actual data |
| DCAT | W3C vocabulary for catalogs; DSP catalogs are DCAT |
| DCP | Decentralized Claims Protocol (formerly IATP) |
| DID / DID document | Decentralized Identifier and the JSON it resolves to (keys + service endpoints) |
| did:web | DID method resolved by fetching `did.json` over HTTPS |
| DIV (formerly DIM) | Commercial wallet used in production, bridged via SAP's open-source SSI DIM Middle Layer |
| DPP | Digital Product Passport: a product's regulated data record, required first for batteries and later for ESPR product groups |
| DPS | Data Plane Signaling: spec for control plane ↔ data plane communication, with SDKs in several languages |
| DSP | Dataspace Protocol: catalog, negotiation and transfer between connectors |
| DTR | Digital Twin Registry |
| EDC | Eclipse Dataspace Components (the framework); also used loosely for a connector |
| EDR | Endpoint Data Reference: endpoint + token for pulling data under an agreement |
| ESMF | Eclipse Semantic Modeling Framework: SDKs and editor for SAMM aspect models |
| ESP / BAP | Enablement Service Provider / Business Application Provider |
| ESPR | EU Ecodesign for Sustainable Products Regulation (EU) 2024/1781, which extends passports to more product groups such as textiles |
| IC-Hub | Industry Core Hub: Python/React orchestrator for providing and consuming twin data |
| IdentityHub | EDC-based DCP wallet for organisations |
| IRS | Item Relationship Service: walks part hierarchies across companies |
| KIT | "Keep It Together": Tractus-X bundle of docs, models and samples per use case |
| ODRL | W3C policy language used for all offers and agreements |
| Participant context | IdentityHub tenant for one identity |
| PCF | Product Carbon Footprint: a product's greenhouse-gas emissions, exchanged under CX-0136 |
| SAMM | Semantic Aspect Meta Model: language for Catena-X aspect models |
| STS | Secure Token Service: signs self-issued ID tokens |
| TCK | Technology Compatibility Kit: conformance tests for DSP and DCP |
| Usage policy | Policy that becomes the legally binding terms of an agreement |
| Usage purpose | Mandatory usage-policy constraint naming why data may be used, e.g. cx.circular.dpp:1 for passports or cx.pcf.base:1 for PCF |
| VC / VP | Verifiable Credential (issuer-signed claim) / Verifiable Presentation (holder-signed bundle of VCs) |
| Virtual Connector (EDC-V) | Upstream EDC runtime that hosts many participants in one deployment; not yet in a Tractus-X release |

## 20. Suggested reading, in order

Read in this order: context first, then the two protocols, then the reference connector and wallet, then your language track and product (3b, 3c), then the Catena-X rules we are certified against. Each phase builds the vocabulary the next one assumes. Items marked *skim* need only their introduction. In week one, run the lab in section 16 alongside Phases 1 and 2.

### Phase 1: Context (why this exists, who runs what)

| # | Read | Focus on |
| --- | --- | --- |
| 1 | [Catena-X operating model: Introduction](https://catenax-ev.github.io/docs/operating-model/why-introduction) | Why a decentralised dataspace, and the vocabulary the rest of the docs use |
| 2 | [Catena-X roles](https://catenax-ev.github.io/docs/operating-model/who-roles-in-the-catena-x-ecosystem) | CSP-A/B, ESP, BAP, DPC; which role we play |
| 3 | [Catena-X service map](https://catenax-ev.github.io/docs/operating-model/what-service-map) | Core vs enablement services; where wallet, BDRS, DTR sit |
| 4 | [EDC adopters manual: overview](https://eclipse-edc.github.io/documentation/for-adopters/) and [Dataspaces](https://eclipse-edc.github.io/documentation/for-adopters/dataspaces/) | Connector, IdentityHub, catalog as components |
| 5 | [Tractus-X KITs](https://eclipse-tractusx.github.io/Kits) (Connector, Digital Twin, Industry Core) *skim* | How Catena-X packages docs per use case |

### Phase 2: Identity and protocols

| # | Read | Focus on |
| --- | --- | --- |
| 6 | [W3C DID Core](https://www.w3.org/TR/did-core/) and [did:web](https://w3c-ccg.github.io/did-method-web/) *skim* | DID document structure, verification methods, services |
| 7 | [W3C VC Data Model](https://www.w3.org/TR/vc-data-model/) *skim* | Issuer, holder, verifier; credentials vs presentations |
| 8 | [EDC IdentityHub manual](https://eclipse-edc.github.io/documentation/for-adopters/identity-hub/) | The clearest prose explanation of the DCP presentation flow |
| 9 | [DSP 2025-1](https://eclipse-dataspace-protocol-base.github.io/DataspaceProtocol/2025-1/) | Model, catalog, negotiation and transfer state machines, HTTPS bindings |
| 10 | [DCP v1.0.1](https://eclipse-dataspace-dcp.github.io/decentralized-claims-protocol/v1.0.1) | Base Identity and Presentation protocols first; issuance later |

### Phase 3: The reference connector and wallet

| # | Read | Focus on |
| --- | --- | --- |
| 11 | EDC adopters: [Control Plane](https://eclipse-edc.github.io/documentation/for-adopters/control-plane/), [Policy Engine](https://eclipse-edc.github.io/documentation/for-adopters/control-plane/policy-engine/), [Data Plane](https://eclipse-edc.github.io/documentation/for-adopters/data-plane/), [Distributions & Deployment](https://eclipse-edc.github.io/documentation/for-adopters/distributions-deployment-operations/) | Entities, scopes, policy functions, runtime assembly |
| 12 | EDC contributors: [Extension Model](https://eclipse-edc.github.io/documentation/for-contributors/runtime/extension-model/), [Dependency Injection](https://eclipse-edc.github.io/documentation/for-contributors/runtime/dependency-injection/), [Programming Primitives](https://eclipse-edc.github.io/documentation/for-contributors/runtime/programming-primitives/), [Entities](https://eclipse-edc.github.io/documentation/for-contributors/control-plane/entities/), [PostgreSQL persistence](https://eclipse-edc.github.io/documentation/for-contributors/postgres-persistence/) | How to write and test an extension |
| 13 | EDC blog: [Data Plane Signaling](https://eclipse-edc.github.io/blog/2026/01/19/data-plane-signaling/), [Management API versions](https://eclipse-edc.github.io/blog/2026/02/19/management-api-upcoming-versions/), [Path to v1](https://eclipse-edc.github.io/blog/2026/03/06/the-path-to-version-1-for-the-eclipse-dataspace-components/), [Data Plane migration](https://eclipse-edc.github.io/blog/2026/05/19/data-plane-migration/) | What is changing in the next 6–12 months |
| 14 | [tractusx-edc docs](https://github.com/eclipse-tractusx/tractusx-edc/tree/main/docs): management API walkthrough, extension READMEs | Catena-X specific behaviour; run the walkthrough end to end |
| 15 | [tractusx-identityhub](https://github.com/eclipse-tractusx/tractusx-identityhub) README, INSTALL.md, docs/architecture | Wallet + issuer deployment and Catena-X specifics |
| 16 | Hands-on: [EDC Minimum Viable Dataspace](https://github.com/eclipse-dataspace-hub/MinimumViableDataspace) and [EDC Samples](https://github.com/eclipse-edc/Samples) | Upstream labs without the Catena-X extensions; do them after the section 16 lab, mainly if you work on EDC internals |

Items 11–13 are concepts everyone needs; the extension and persistence pages in item 12 matter only to the JVM track.

### Phase 3b: Your language track (pick one or two)

| Track | Read and run |
| --- | --- |
| All tracks | [Industry Core Hub README and architecture](https://github.com/eclipse-tractusx/industry-core-hub) (the full provider and consumer flow in one codebase), [api-hub](https://github.com/eclipse-tractusx/api-hub) OpenAPI specs, [tractus-x-umbrella](https://github.com/eclipse-tractusx/tractus-x-umbrella) to run a local dataspace, and the [Tractus-X changelog](https://eclipse-tractusx.github.io/blog-changelog) for component versions and known issues per release |
| Python | [Tractus-X SDK docs](https://eclipse-tractusx.github.io/tractusx-sdk/main/), the IC-Hub `ichub-backend/`, [tractusx-testlab](https://github.com/eclipse-tractusx/tractusx-testlab), the [ESMF Python aspect-model loader](https://github.com/eclipse-esmf/esmf-sdk-py-aspect-model-loader) |
| TypeScript/JS | The IC-Hub `ichub-frontend/`, [tractusx-edc-dashboard](https://github.com/eclipse-tractusx/tractusx-edc-dashboard), [portal-frontend](https://github.com/eclipse-tractusx/portal-frontend) |
| C#/.NET | [portal-backend](https://github.com/eclipse-tractusx/portal-backend), [ssi-credential-issuer](https://github.com/eclipse-tractusx/ssi-credential-issuer) |
| Go, Rust or any custom data plane | [Data Plane Signaling spec and SDKs](https://github.com/eclipse-dataplane-signaling), [EDC data plane migration post](https://eclipse-edc.github.io/blog/2026/05/19/data-plane-migration/) |
| JVM (Java/Kotlin) | Items 12–15, then the source of [DTR](https://github.com/eclipse-tractusx/sldt-digital-twin-registry), [BDRS](https://github.com/eclipse-tractusx/bpn-did-resolution-service) and [IRS](https://github.com/eclipse-tractusx/item-relationship-service) |
| Digital twins (any language) | [CX-0002](https://catenax-ev.github.io/docs/standards/CX-0002-DigitalTwinsInCatenaX), the [AAS API profiles (IDTA-01002 v3.2)](https://industrialdigitaltwin.io/aas-specifications/IDTA-01002/v3.2/http-rest-api/service-specifications-and-profiles.html), [sldt-semantic-models](https://github.com/eclipse-tractusx/sldt-semantic-models) |

### Phase 3c: Your product (alongside 3b)

| Product | Read and run |
| --- | --- |
| Connector and Dataspace as a Service | [0.13 migration guide](https://github.com/eclipse-tractusx/tractusx-edc/blob/main/docs/migration/2026_06-Version_0.12.x_0.13.x.md), [Connector KIT operation view](https://eclipse-tractusx.github.io/docs-kits/kits/connector-kit/operation-view), [IdentityHub DCP API walkthrough](https://github.com/eclipse-tractusx/tractusx-identityhub/blob/main/docs/usage/dcp-api-walkthrough/README.md) (run your own issuer), [multi-dataspace decision record](https://github.com/eclipse-tractusx/tractusx-edc/blob/main/docs/development/decision-records/2026-01-07_multi-dataspace-support/README.md), [umbrella observability guide](https://github.com/eclipse-tractusx/tractus-x-umbrella/blob/main/docs/user/common/guides/observability/observability.md) |
| Battery passport | [CX-0160 Battery Passport base](https://catenax-ev.github.io/docs/next/standards/CX-0160-BatteryPassport/CX-0160-BatteryPassport-base) (preview until 24 November 2026), [CX-0143 Digital Product Passport](https://catenax-ev.github.io/docs/standards/CX-0143-UseCaseCircularEconomyDigitalProductPassportStandard/introduction), [Eco Pass KIT](https://eclipse-tractusx.github.io/docs-kits/kits/eco-pass-kit/adoption-view), [EU Battery Regulation](https://eur-lex.europa.eu/eli/reg/2023/1542/oj) |
| PCF | [CX-0136 Use Case PCF](https://catenax-ev.github.io/docs/standards/CX-0136-UseCasePCF), [PCF Exchange KIT](https://eclipse-tractusx.github.io/docs-kits/kits/product-carbon-footprint-exchange-kit/adoption-view), [PCF Data Acquisition KIT](https://eclipse-tractusx.github.io/docs-kits/kits/pcf-data-acquisition-kit/adoption-view), [PCF Rulebook](https://catenax-ev.github.io/docs/rulebooks/CX-NFR-PCF) |
| Textile passport | [ESPR](https://eur-lex.europa.eu/eli/reg/2024/1781/oj), [CX-0143 aspect models](https://catenax-ev.github.io/docs/standards/CX-0143-UseCaseCircularEconomyDigitalProductPassportStandard/CX-0143-SEM) (the generic passport modules), [cosh.eco on the textile timeline](https://cosh.eco/en/articles/eu-digital-product-passport-espr-explained-fashion-textiles) |

### Phase 4: Catena-X normative rules

| # | Read | Focus on |
| --- | --- | --- |
| 17 | [CX-0018 Dataspace Connectivity v4.2](https://catenax-ev.github.io/docs/standards/CX-0018-DataspaceConnectivity) | MUSTs for any Catena-X connector: DSP/DCP versions, catalog conventions |
| 18 | [CX-0149 Wallet Requirements v2.0.0](https://catenax-ev.github.io/docs/standards/CX-0149-WalletRequirements), [CX-0049 DID Document](https://catenax-ev.github.io/docs/standards/CX-0049-DIDDocumentSchema), [CX-0050 Credentials](https://catenax-ev.github.io/docs/standards/CX-0050-CXSpecificCredentials/CX-0050-CatenaXSpecificVerifiableCredentials) | What our wallet must do; exact credential schemas |
| 19 | [CX-0152 Policy Constraints](https://catenax-ev.github.io/docs/standards/CX-0152-PolicyConstrainsForDataExchange) and its [Appendix](https://catenax-ev.github.io/docs/standards/CX-0152-PolicyConstrainsForDataExchange/Appendix) | Allowed constraints, values and samples |
| 20 | [CX-0002 Digital Twins](https://catenax-ev.github.io/docs/standards/CX-0002-DigitalTwinsInCatenaX), [CX-0003 SAMM](https://catenax-ev.github.io/docs/standards/CX-0003-SAMMSemanticAspectMetaModel), [CX-0053 Discovery](https://catenax-ev.github.io/docs/standards/CX-0053-BPNDiscoveryServiceAPIs), [CX-0151 Industry Core](https://catenax-ev.github.io/docs/standards/CX-0151-IndustryCoreBasics) | The data layer on top of EDC |

### Phase 5: Staying current (bookmark, revisit monthly)

| # | Read | Focus on |
| --- | --- | --- |
| 21 | [tractusx-edc releases](https://github.com/eclipse-tractusx/tractusx-edc/releases) | Breaking changes and migration guides |
| 22 | [EDC TCK compatibility reports](https://eclipse-edc.github.io/documentation/compatibility/) | Which DSP/DCP versions upstream passes |
| 23 | [Catena-X release notes](https://catenax-ev.github.io/blog-releasenotes) and the [tractusx-dev mailing list](https://www.eclipse.org/lists/tractusx-dev/) | Release trains and community decisions |
| 24 | [EDC Connector releases](https://github.com/eclipse-edc/Connector/releases) and [IdentityHub releases](https://github.com/eclipse-edc/IdentityHub/releases) | What reaches us with the next tractusx-edc or tractusx-identityhub version bump |

### Where to ask when stuck

- **First contact with Tractus-X:** the [tractusx-dev mailing list](https://www.eclipse.org/lists/tractusx-dev/), which the community page names as the entry point ([community intro](https://eclipse-tractusx.github.io/community/intro)).
- **Quick questions:** the Tractus-X [Eclipse Matrix space](https://matrix.to/#/#automotive.tractusx:matrix.eclipse.org).
- **Product questions:** the office hours and product weeklies on the [open meetings page](https://eclipse-tractusx.github.io/community/open-meetings), e.g. the IdentityHub weekly.
- **Bugs:** an issue on the repo that owns the behaviour; section 14's upstream table says whether that is the Tractus-X repo or upstream EDC.

Sources for this document are linked inline in each section. As of 10 October 2026.
