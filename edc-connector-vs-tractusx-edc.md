# Eclipse EDC Connector vs. Eclipse Tractus-X EDC — Engineering Comparison and Selection Guide

**Version 2.0 · 10 October 2026**

## 1. Executive summary

`eclipse-edc/Connector` and `eclipse-tractusx/tractusx-edc` are not competing products. Tractus-X EDC
is a downstream *distribution* built on the upstream EDC *framework*. Of the production code that
Tractus-X EDC 0.13.0 ships, about **81% is upstream EDC** (192 modules from EDC 0.17.0) and **19% is
Tractus-X's own** (about 25,000 lines). The real choice is between adopting a finished,
Catena-X-specific distribution and assembling a distribution of your own from the framework.

> **Bottom line.** If you are joining Catena-X, or must interoperate with Catena-X connectors, use
> **Tractus-X EDC** — it is the conformance reference and nothing else implements its Catena-X
> policy, identity and protocol behaviour. For any other dataspace, build on **upstream EDC's 1.0
> line**, choosing the classic or virtual (multi-tenant) flavour to fit, and use Tractus-X's
> packaging as a blueprint.

**Key findings**

| # | Finding | Implication |
|---|---|---|
| 1 | Upstream is in 1.0 release candidates (latest **v1.0.0-RC4**, 29 Sep 2026); 1.0 final is not yet released. 1.0 collapses 45 SPI modules to 3 and 44 library modules to 5, deleting published Maven coordinates. | Starting on the 0.18.x line now means paying a large migration later. New upstream projects should start on 1.0. |
| 2 | Upstream ships **two connector flavours**. CEL policy expressions, Management API v5, partner groups and multi-tenancy exist only in the *virtual* flavour. | "Upstream has feature X" must always be qualified by flavour. Tractus-X is architecturally a classic-flavour distribution. |
| 3 | Tractus-X 0.13.0 (27 Aug 2026) runs on EDC 0.17.0. The age of the upstream pin at each Tractus-X feature release has grown every cycle: **14 → 43 → 82 → 129 days**. No branch or commit shows EDC 1.0 adoption has started. | Expect a long gap before Tractus-X carries 1.0 features, and a disruptive upgrade when it does. |
| 4 | Upstream has **deprecated its EDR cache**; EDR lifecycle moves into the data plane under Data Plane Signaling. Tractus-X's consumer-pull model depends on its own EDR API with automatic token refresh. | The projects are diverging on a core consumer-side mechanism. |
| 5 | Upstream ships **no runnable artefact** — no images, no charts, no versioned database migrations. Tractus-X ships multi-arch images, two Helm charts and 76 Flyway migrations. | Choosing upstream means owning delivery engineering. |
| 6 | Both projects rely on a narrow maintainer core. Over the last 12 months upstream's top three authors wrote **93%** of human commits; Tractus-X's top three wrote **71%**, from a wider pool of 27 authors. | Contributor concentration is a long-term risk for either choice, and greater upstream. |

## 2. Scope and method

**Repositories and versions examined**

| Project | Versions examined | Why |
|---|---|---|
| `eclipse-edc/Connector` | v0.17.0 · v0.18.1 · v1.0.0-RC1 · v1.0.0-RC4 · `main` (`1.0.0-SNAPSHOT`, 9 Oct 2026) | v0.17.0 is what Tractus-X consumes; v0.18.1 is the latest stable; RC4 is the latest pre-release |
| `eclipse-tractusx/tractusx-edc` | 0.10.0 – 0.13.0 tags · `main` (`0.14.0-SNAPSHOT`, 9 Oct 2026) | 0.13.0 is the latest stable; earlier tags are used for the lag analysis |

**Method.** Every figure in this document was derived from source trees at the stated release tags,
from git history, or from published release notes and decision records. Module counts come from
`settings.gradle.kts`; lines of code (LOC) count Java files under `src/main`; API versions come from
JAX-RS `@Path` annotations; contributor metrics exclude bot accounts. Repository star and fork counts
are omitted: they carry little decision value and could not be read consistently on the same day.
Statements of judgement and estimates are labelled as such.

**Conventions.** *Upstream* means `eclipse-edc/Connector`. *Tractus-X* means
`eclipse-tractusx/tractusx-edc`. A glossary of abbreviations is in Appendix C.

## 3. How the two projects relate

```
          eclipse-edc/Connector                  upstream framework
          SPI + core + extensions                Maven artefacts and BOMs only;
          published to Maven Central             no runnable image
                    │
                    │  Tractus-X pins edc = "0.17.0" and composes
                    │  five upstream BOMs
                    ▼
          eclipse-tractusx/tractusx-edc          downstream distribution
          upstream BOMs + Catena-X extensions    Docker images, Helm charts,
          + exclusions                           Flyway migrations
```

Tractus-X composes `controlplane-base-bom`, `controlplane-dcp-bom`, `controlplane-feature-sql-bom`,
`dataplane-base-bom` and `dataplane-feature-sql-bom` at version 0.17.0. The contract-negotiation and
transfer-process state machines, the policy engine, the DSP 2025-1 protocol engine, the DCP credential
verifier and the data plane framework that Tractus-X runs are all upstream code.

Tractus-X contributes Catena-X policy functions; Business Partner Number (BPN) identity and
BPN-to-DID resolution; integration with the DIV wallet; a binding for the legacy DSP 0.8 protocol;
EDR and token-refresh APIs; agreement retirement; and the operational packaging — images, charts and
database migrations.

**Measured composition of Tractus-X EDC 0.13.0**

| Source | Modules | Production Java LOC | Share |
|---|---|---|---|
| Upstream EDC 0.17.0, reachable from the five BOMs, minus modules Tractus-X excludes | 192 | ~106,000 | 81% |
| Tractus-X's own code (excluding test modules) | 117 | ~25,000 | 19% |

*Method: transitive closure of Gradle project dependencies from the five BOMs. Third-party libraries
and upstream's separate Technology repositories (AWS, Azure) are not counted.*

The question this document answers is therefore:

> Do I need a Catena-X-compliant, batteries-included, operable distribution — or do I need a
> framework that I will assemble into my own distribution?

## 4. Project facts

| Attribute | Upstream EDC | Tractus-X EDC |
|---|---|---|
| Governance | Eclipse Foundation, Eclipse Dataspace Components project | Eclipse Foundation, Eclipse Tractus-X project; requirements driven by the Catena-X association |
| Role | Framework / SDK | Product distribution |
| Latest stable release | v0.18.1 (25 Sep 2026) | 0.13.0 (27 Aug 2026) |
| Latest pre-release | v1.0.0-RC4 (29 Sep 2026) | none open |
| `main` version | `1.0.0-SNAPSHOT` | `0.14.0-SNAPSHOT` |
| Upstream EDC consumed | — | 0.17.0 (released 20 Apr 2026) |
| Gradle modules | 310 at v0.18.1 · 213 at v1.0.0-RC4 | 117 at 0.13.0 |
| Production Java LOC | ~138,000 at v0.18.1 · ~132,000 at v1.0.0-RC4 | ~25,000 at 0.13.0 |
| Build | Gradle Kotlin DSL, `org.eclipse.edc.edc-build` 1.5.2 | Same plugin and version |
| CI build JDK | 17 (Temurin) | 21 (Temurin) |
| Runtime image | none published | `eclipse-temurin:25-jre-alpine`, amd64 and arm64 |
| Helm charts | none | `tractusx-connector` (19 templates), `tractusx-connector-memory` (10 templates) |
| Database schema handling | DDL files plus `edc.sql.schema.autocreate` for an empty database; no versioned migrations | 76 Flyway migration scripts, applied at start-up |
| Licence | Apache-2.0 | Apache-2.0 (code), CC-BY-4.0 (non-code) |
| History | First commit Jan 2021; 3,315 commits on `main` | First commit Mar 2022; 2,235 commits on `main` |

**Community and activity** (12 months to 10 Oct 2026, bots excluded)

| Metric | Upstream EDC | Tractus-X EDC |
|---|---|---|
| Human commits | 375 | 170 |
| Commits in the last 90 days (all authors) | 128 | 100 |
| Distinct human authors | 15 | 27 |
| Authors with five or more commits | 4 | 5 |
| Share of human commits by top author | 44% | 33% |
| Share of human commits by top three authors | 93% | 71% |
| Distinct human authors, all time | 99 | 71 |

Upstream has roughly twice the commit throughput but is carried by three people. Tractus-X draws on a
wider but thinner pool. Neither has a deep bench.

Recent Tractus-X activity is mostly maintenance: of 43 commits on `main` since 30 August 2026, 34 are
chores (predominantly dependency updates). The only feature work is a Gateway API `HTTPRoute`
template in the Helm chart and test-infrastructure changes. `main` still pins EDC 0.17.0.

## 5. Repository structure

### 5.1 Upstream EDC (1.0 line)

```
spi/               core-spi, control-plane-spi, decentralized-claims-spi
core/              runtime, control plane, policy monitor, catalog crawler,
                   libraries (core-lib, control-plane-lib)
extensions/        common/ (iam, sql, http, api, auth, crypto, vault, nats,
                     events, otel ...), control-plane/ (management API v4 and
                     v5, SQL stores, callbacks, NATS task executors),
                   federated-catalog/, policy-monitor/
data-protocols/    dsp/ (DSP 2025-1, classic and virtual bindings, dsp-lib),
                   data-plane-signaling/ (client, OAuth2, token exchange)
dist/bom/          11 BOMs — the actual consumption surface
system-tests/      e2e, TCKs (DSP, DCP, DPS), telemetry, version API
docs/developer/    decision records — the authoritative design documentation
```

There is no `Dockerfile` and no launcher module. Upstream publishes Maven artefacts and BOMs;
assembling a runnable connector is the adopter's job.

### 5.2 The 1.0 module consolidation

| | v0.17.0 | v0.18.1 | v1.0.0-RC1 | v1.0.0-RC4 | `main` |
|---|---|---|---|---|---|
| Gradle modules | 299 | 310 | 210 | 213 | 212 |
| SPI modules | 45 | 45 | 3 | 3 | 3 |
| Library (`*-lib`) modules | 44 | 44 | 5 | 5 | 4 |
| BOMs | 8 | 13 | 11 | 11 | 11 |
| Data plane BOMs | 2 | 2 | 0 | 0 | 0 |

Two decision records dated 18 June 2026 define the change, and it first appears in v1.0.0-RC1:

- **SPI consolidation.** About 45 fine-grained SPI modules become a layer-oriented set: `core-spi`
  (what you need to build a runtime), `control-plane-spi` (what you need to build a connector),
  `data-plane-spi`, and one module per protocol (`decentralized-claims-spi`, `signaling-spi`,
  `dataspace-protocol-spi`). Three of these live in the Connector repository.
- **Library consolidation.** About 43 `*-lib` modules become `core-lib`, `control-plane-lib`, `dsp-lib`
  (which absorbs a 13-module matrix), `jsonld-lib` and a standalone `jws2020-lib`.

Both records state that published Maven coordinates disappear and that this is a breaking change for
adopters who depend on individual modules. The layout is still moving after the last release
candidate: on `main`, `jsonld-lib` has since been merged into `core-lib`. Do not treat RC4's module
list as final.

### 5.3 Upstream ships two connector flavours

From 0.18.0 the BOMs define two distinct runtimes:

| | Classic connector | Virtual connector |
|---|---|---|
| BOMs | `controlplane-base-bom`, `-dcp-bom`, `-feature-sql-bom`, `-feature-dcp-bom` | `controlplane-virtual-base-bom`, `-feature-sql-bom`, `-feature-dcp-bom`, `-feature-nats-bom` |
| Tenancy | Single participant | Many participants per runtime (participant contexts) |
| Management API | v4 (v3 also on 0.x) | v5, with OAuth2 authentication and scope-based authorization |
| CEL policy expressions | Not included | Included |
| Partners and partner groups | Not included | Included (v1.0.0-RC4) |
| Dataspace profiles | Single | Multiple concurrently, selected per participant |
| Task execution | In-process state machines | Optional NATS-based task distribution |

New upstream capabilities are landing predominantly in the virtual flavour. Tractus-X is built on the
classic flavour, so adopting upstream 1.0 would not by itself bring CEL, v5 or partner groups to
Tractus-X; that would require a further architectural move to the virtual flavour.

### 5.4 The data plane leaves the repository

The in-repository Data Plane Framework was deprecated in v0.16.0 but continued to ship: every 0.x
release through v0.18.1 includes `dataplane-base-bom` and `dataplane-feature-sql-bom`. Both are removed
in v1.0.0-RC1. On the 1.0 line, data planes are built separately — upstream's own tests use the
`org.eclipse.dataplane-core` SDK (`eclipse-dataplane-core/dataplane-sdk-java`; version 1.1.1 at RC4,
1.2.1 on `main`) — and the control plane talks to them through **Data Plane Signaling (DPS)**, a
protocol specified at `eclipse-dataplane-signaling/dataplane-signaling`. DPS became the default
transfer mechanism in v0.18.0.

Tractus-X 0.13.0, on 0.17.0, still runs the in-repository data plane framework. Custom `DataSource`
and `DataSink` implementations are the code most exposed to this change.

### 5.5 Tractus-X EDC

```
spi/                     callback, edr, tokenrefresh, bdrs-client, dataflow,
                         dcp, did-document-service, dsp-spi-08
core/                    edr-core, json-ld-core, json-ld-cx, core-utils
edc-extensions/          the Catena-X delta (Section 7)
edc-controlplane/        edc-controlplane-base
                         edc-controlplane-postgresql-hashicorp-vault   production image
                         edc-runtime-memory                           development image
edc-dataplane/           edc-dataplane-base
                         edc-dataplane-hashicorp-vault                production image
charts/                  tractusx-connector, tractusx-connector-memory
edc-tests/               e2e suites, DSP and DCP TCKs, compatibility, upgradeability
docs/                    migration guides (one per version pair), Management API
                         walkthrough, decision records
```

`edc-controlplane/edc-controlplane-base/build.gradle.kts` is the distribution definition. It composes
the upstream BOMs, adds 32 Tractus-X modules, and explicitly excludes upstream modules
(Section 8).

## 6. Architecture and runtime

| Concern | Upstream EDC | Tractus-X EDC |
|---|---|---|
| Consumption model | Maven coordinates and BOMs | OCI images and Helm charts (Maven artefacts also published) |
| Runtime assembly | Write a launcher, select modules, build an image | Pre-assembled; configure through Helm values and environment variables |
| Control plane | Classic or virtual flavour (Section 5.3) | `edc-controlplane-postgresql-hashicorp-vault` (fixed composition) |
| Data plane | In-repo framework on 0.x; external, DPS-integrated on 1.0 | `edc-dataplane-hashicorp-vault` (0.17.0 framework) |
| Persistence | In-memory or PostgreSQL; schema created on an empty database; upgrades of an existing schema are the adopter's job | PostgreSQL required in production; Flyway migrations run at start-up |
| Secrets | Pluggable `Vault` SPI — HashiCorp Vault in the Connector repository; Azure Key Vault in `Technology-Azure` | HashiCorp Vault only; images bundling Azure Key Vault were discontinued |
| Multi-tenancy | Virtual connector (participant contexts since 0.15.0; virtual BOMs since 0.18.0) | Single tenant; a mandatory participant context ID exists as preparation |
| Eventing | Callbacks, event dispatcher, NATS event publishing, OpenTelemetry logger | Callbacks, OpenTelemetry collector event subscriber, log4j2 JSON logging |
| Deployment target | Whatever you build | Kubernetes |

### 6.1 External runtime dependencies

Neither repository contains everything a working connector needs. The services below must exist
alongside it.

| Dependency | Upstream (DCP deployment) | Tractus-X |
|---|---|---|
| Credential service / wallet | A DCP 1.0 credential service — typically Eclipse EDC IdentityHub, which also hosts the Secure Token Service (moved there in 0.14.0) | A DIV tenant (the SAP-operated wallet used in Catena-X) holding the participant's Catena-X credentials, plus an OAuth client for its STS |
| Identifier resolution | DID resolution (`did:web`) | DID resolution plus the BPN-DID Resolution Service (BDRS, `eclipse-tractusx/bpn-did-resolution-service`) |
| Trust configuration | Trusted credential issuers | Trusted issuers (`dcp.trustedIssuers`) |
| Database | PostgreSQL (or in-memory for development) | PostgreSQL; the chart can install PostgreSQL 18 |
| Secret store | Any `Vault` implementation | HashiCorp Vault (known incompatible: 1.18.1) |
| Participant identity | A DID | A DID (`participant.id`), a BPNL (`participant.bpnl`), and an immutable participant context ID |

## 7. What Tractus-X adds

### 7.1 Identity, trust and business-partner semantics

| Extension | Purpose |
|---|---|
| `bpn-validation` (API, SPI, core, SQL store) | BPNL validation and a business-partner-group store with a CRUD API — the core Catena-X access-control primitive |
| `bdrs-client` | Client for BDRS, mapping BPNLs to DIDs. No upstream equivalent |
| `dcp:tx-dcp`, `dcp:cx-dcp` | DCP wiring; `cx-dcp` holds Catena-X default credential scopes and scope extractors |
| `dcp:tx-dcp-sts-div` | Secure Token Service client for DIV |
| `dcp:verifiable-presentation-cache` | Caches verifiable presentations to reduce wallet round-trips |
| `did-document:*` | Registers the connector's DSP endpoint as a service entry in its own DID document |
| `connector-discovery:*` | Resolves a counterparty's connector endpoint and DSP version from its identifier |

### 7.2 Policy — the largest single differentiator

Tractus-X implements the Catena-X policy vocabulary defined by standard CX-0152. In 0.13.0 the
`cx-policy` extension registers **28 Catena-X policy left-operands**, implemented by **32
constraint-function classes**, with **63 registrations** across four policy scopes (catalog, contract
negotiation, transfer process and policy monitor):

| Group | Left-operands |
|---|---|
| Access and identity (7) | `Membership`, `FrameworkAgreement`, `Dismantler`, `BusinessPartnerNumber`, `BusinessPartnerGroup`, `AffiliatesBpnl`, `AffiliatesRegion` |
| Usage (4) | `UsagePurpose`, `UsageRestriction`, `ExclusiveUsage`, `DataFrequency` |
| Time limits (5) | `DataUsageEndDate`, `DataUsageEndDurationDays`, `DataUsageEndDefinition`, `DataProvisioningEndDate`, `DataProvisioningEndDurationDays` |
| Contract terms (12) | `ContractReference`, `ContractTermination`, `Precedence`, `VersionChanges`, `Warranty`, `WarrantyDefinition`, `WarrantyDurationMonths`, `Liability`, `JurisdictionLocation`, `JurisdictionLocationReference`, `ConfidentialInformationMeasures`, `ConfidentialInformationSharing` |

JSON-schema validators reject non-conformant policies at creation time rather than at evaluation time;
in 0.13.0 this validation is available only through Management API v4. A separate `cx-policy-legacy`
extension keeps pre-CX-0152 policies working.

Upstream provides the ODRL policy engine, evaluation-plan and validation APIs, and — in the virtual
flavour — CEL expressions and partner groups, but no industry-specific constraint functions. Partners
and partner groups (v1.0.0-RC4) let a policy ask "which group is this counterparty in?" without Java
code, which covers part of what `BusinessPartnerGroup` does in Tractus-X. Nothing upstream replicates
the CX-0152 vocabulary. Reimplementing it is a substantial effort with real correctness risk; if your
dataspace does not need it, it is dead weight and a source of policy rejections.

### 7.3 Data plane and transfer

| Extension | Purpose |
|---|---|
| `dataplane-proxy:edc-dataplane-proxy-consumer-api` | Consumer-side proxy, so business applications call a local endpoint instead of handling EDRs themselves |
| `dataplane-proxy:dataplane-proxy-http`, `dataplane-public-api-v2` | Provider-side HTTP proxying, optionally passing through the origin's status code |
| `dataplane-token-refresh:*`, `tokenrefresh-handler` | Automatic EDR token refresh |
| `non-finite-provider-push:*` | Keeps a push transfer open for repeated pushes |
| `dataplane:kafka:*` | Kafka transfers with OAuth2 and broker-level access revocation through Kafka ACLs. Released in 0.13.0 but opt-in: not part of the published images |
| `dataplane:dataflow:*` | Data-flow inspection API |

### 7.4 Contract lifecycle

| Extension | Purpose |
|---|---|
| `agreements:retirement-evaluation-*` | Retire a contract agreement, ending access under an agreement that is otherwise still valid. No upstream equivalent |
| `agreements-bpns:*` | Records counterparty BPNs against each agreement for audit and reporting |
| `edr:edr-api-v2`, `edr-callback`, `edr-index-lock-sql` | EDR management API, replacing upstream's `edr-cache-api` (which Tractus-X excludes) |

### 7.5 Protocol

Upstream ships only DSP 2025-1; older versions were removed in v0.16.0. Tractus-X carries a complete
**DSP 0.8 binding in its own repository** — 13 modules covering catalog, negotiation and transfer, with
HTTP APIs, transformers and a dispatcher — so that Catena-X connectors can still talk to participants
that have not migrated. Tractus-X therefore serves DSP 0.8 and DSP 2025-1 side by side, with
connector-side version negotiation. It dropped DSP 2024-1 in 0.12.0.

### 7.6 Operations

Flyway migration modules (76 scripts) · `log4j2-monitor` (JSON structured logging) ·
`event-subscriber` (domain events to an OpenTelemetry collector) · `token-interceptor` ·
`single-participant-vault` · validators for contract-definition policies and empty asset selectors ·
`data-flow-properties-provider`.

## 8. What Tractus-X removes or restricts

| Removed or restricted | Reason given | Consequence |
|---|---|---|
| Federated catalog (`federated-catalog-*`) | "Not proven to be usable" | 0.13.0 drops the `edc_federated_catalog` table through a migration unless `tx.edc.postgresql.migration.federatedcatalog.enabled=false` is set beforehand. Upstream, by contrast, ships three federated-catalog BOMs |
| Upstream `edr-cache-api` | Signature clash with Tractus-X's EDR API | Replaced by `/v3/edrs` with automatic token refresh |
| `decentralized-claims-sts-remote-client` | Replaced by the DIV-aware STS client | Wallet integration is DIV-shaped by default |
| `data-plane-util` | Transitive dependency conflict | None for operators |
| Azure Key Vault distributions | Discontinued (decision record, Oct 2024) | HashiCorp Vault is the only supported secret store in published images |
| DSP 2024-1 | Removed in 0.12.0 | DSP 0.8 and 2025-1 only |
| Mock connector runtime | Removed in 0.13.0 | Use the in-memory runtime for development |

Tractus-X also does not ship upstream's virtual (multi-tenant) connector, CEL expressions, Management
API v5, partner groups or NATS task distribution.

## 9. Capability matrix

Legend: **Yes** — present · **Partial** — present with a material limitation · **No** — absent ·
**DIY** — you build it.

| Capability | Upstream v0.18.1 | Upstream v1.0.0-RC4 | Tractus-X 0.13.0 |
|---|---|---|---|
| **Protocol** | | | |
| DSP 2025-1 | Yes | Yes | Yes |
| DSP 0.8 | No | No | Yes — in-repo binding |
| DSP TCK in CI | Yes | Yes | Yes |
| DPS (Data Plane Signaling) TCK in CI | Yes | Yes | No |
| Multiple dataspace profiles per runtime | Yes — virtual flavour | Yes — virtual flavour | No |
| **Identity and trust** | | | |
| DCP 1.0 | Yes | Yes | Yes |
| DCP TCK in CI | Yes | Yes | Yes |
| VCDM 2.0 credential verification | Yes | Yes | Yes — inherited from 0.17.0 |
| BPN-to-DID resolution (BDRS) | No | No | Yes |
| DIV wallet STS client | No | No | Yes |
| Verifiable-presentation caching | No | No | Yes |
| DID-document self-registration | No | No | Yes |
| Connector discovery | Yes — v5beta, virtual flavour | Yes — v5, virtual flavour | Yes — v3 (v4alpha deprecated) |
| **Policy** | | | |
| ODRL engine, evaluation plan, validation API | Yes | Yes | Yes |
| CEL policy expressions | Yes — virtual flavour | Yes — virtual flavour | No |
| Partners and partner groups | No | Yes — virtual flavour | Partial — BPN groups, Catena-X model |
| Industry constraint functions | DIY | DIY | Yes — 28 CX-0152 left-operands |
| Policy validation at creation | Yes — configurable JSON schemas | Yes — configurable JSON schemas | Yes — CX schemas, Management API v4 only |
| **Contract and transfer** | | | |
| Agreement retirement | No | No | Yes |
| Non-finite provider push | Yes | Yes | Yes |
| Data Plane Signaling | Yes — default | Yes — the data plane integration | Partial — client present, not the default |
| In-repository data plane framework | Partial — deprecated | No — external SDK | Yes |
| EDR management | Partial — `edr-cache-api` v3, deprecated | No — moved into the data plane under DPS | Yes — `/v3/edrs` with auto-refresh |
| Consumer-side data plane proxy | No | No | Yes |
| Kafka transfer | Partial — `data-plane-kafka`, documented as not production-grade | No | Partial — OAuth2 and ACL revocation; opt-in |
| S3 and Azure Blob transfer | Yes — via Technology repositories | Yes — via Technology repositories | Yes — in published images |
| **Management API** | | | |
| v3 | Yes | No | Yes |
| v4 | Yes | Yes — classic flavour | Yes — inherited from 0.17.0 |
| v5 with scope-based authorization | Partial — `/v5beta`, virtual flavour | Yes — virtual flavour | No |
| BPN-group, agreement-retirement and data-flow APIs | No | No | Yes |
| **Operations** | | | |
| Published Docker images | No | No | Yes — amd64 and arm64 |
| Helm charts | No | No | Yes |
| Versioned database migrations | No — create-on-empty only | No — create-on-empty only | Yes — 76 Flyway scripts |
| Upgradeability tests in CI | No | No | Yes |
| Multi-tenancy | Yes — virtual flavour | Yes — virtual flavour | No |
| NATS task distribution and events | Yes | Yes | No |
| Structured JSON logging | Partial — OpenTelemetry logger | Partial — OpenTelemetry logger | Yes — log4j2 |
| Federated catalog | Yes | Yes | No — removed |

## 10. Management API and integration surface

Both projects expose the EDC Management API — assets, policy definitions, contract definitions,
catalog, negotiations, transfer processes and agreements — as JSON-LD over HTTP. The version lines have
separated:

| Release | Versions served |
|---|---|
| Upstream v0.17.0 / v0.18.1 | `/v3` and `/v4` for the full resource set (`/v3/edrs` only on v3); `/v5beta` in the virtual flavour |
| Upstream v1.0.0-RC4 | `/v4` (classic flavour; adds `/v4/dataplanes`); `/v5` (virtual flavour, participant-scoped: `/v5/participants/{id}/…`, plus CEL expressions, DCP scopes, dataspace profiles, partners and partner groups). `/v3` and the EDR endpoints are gone |
| Tractus-X 0.13.0 | Upstream `/v3` and `/v4` inherited from 0.17.0, plus its own `/v3/business-partner-groups`, `/v3/edrs`, `/v3/contractagreements/retirements`, `/v3/connectordiscovery`, `/v3/dataflows`; the last two also on deprecated `/v4alpha` |

Points that matter for client code:

- **Tractus-X serves three generations at once** (v3, v4 and deprecated v4alpha). Its 0.13.0 release
  notes state that the management-API parts inherited from upstream are marked deprecated in the
  published OpenAPI specification, that Tractus-X cannot change that downstream, and that its strategy
  for the management-API switch had not been decided at release time.
- **Upstream's next API is tied to the virtual flavour.** v5 is participant-scoped and requires OAuth2
  with scopes; moving from v4 to v5 is a change of security model, not just of paths.
- **Isolate API versions behind a client abstraction** on either side; do not hard-code version paths.

Tractus-X's Management API walkthrough (`docs/usage/management-api-walkthrough/`, ten chapters) is the
best practical tutorial in either repository and is useful even for upstream deployments.

## 11. Build, dependencies and quality engineering

| | Upstream EDC | Tractus-X EDC |
|---|---|---|
| Build | Gradle Kotlin DSL, `edc-build` 1.5.2 | Same |
| CI JDK | 17 | 21 |
| Version catalogues | `gradle/libs.versions.toml` | Plus `libs.stable.versions.toml`, pinning a released Tractus-X/EDC pair for compatibility tests |
| Test types | Unit, component, e2e, DSP/DCP/DPS TCKs, nightly, BOM smoke tests | Unit, 12 e2e suites, DSP and DCP TCKs, compatibility (released versus snapshot), upgradeability, Helm lint, deployment test |
| Security and supply chain in CI | CodeQL; pull-request dependency review with a licence allow-list | CodeQL, Trivy, KICS, TruffleHog secret scanning, zizmor and poutine workflow linting, Dependabot, SonarCloud, Eclipse `DEPENDENCIES` IP file |
| Test reporting | GitHub Actions | Allure reports published per release |

Two Tractus-X practices are worth adopting whichever option you choose:

1. **Compatibility tests** boot a released runtime against a snapshot runtime to catch cross-version
   protocol breaks. Note that `libs.stable.versions.toml` on `main` still pins 0.12.1 / EDC 0.15.1, one
   release behind.
2. **Upgradeability tests** perform a full Helm upgrade from the previous release, exercising the
   Flyway migrations.

Tractus-X also maintains the Eclipse Foundation IP paperwork (per-dependency approvals,
`LICENSE_non-code`). If your organisation has an open-source compliance gate, that work is done.

## 12. Release cadence, versioning and upstream lag

### 12.1 Upstream

| Release | Date |
|---|---|
| v0.13.0 | 21 May 2025 |
| v0.14.0 | 19 Aug 2025 |
| v0.15.0 / v0.15.1 | 2 Dec 2025 / 17 Dec 2025 |
| v0.16.0 | 19 Feb 2026 |
| v0.17.0 | 20 Apr 2026 |
| v0.18.0 | 30 Jun 2026 |
| v0.16.1 (patch on an older line) | 11 Aug 2026 |
| v1.0.0-RC1 | 7 Sep 2026 |
| v1.0.0-RC2, RC3 | 16 Sep 2026 |
| v0.18.1 | 25 Sep 2026 |
| v1.0.0-RC4 | 29 Sep 2026 |

Minor releases have arrived every two to three months, with patch releases on older lines. There is no
0.19.0: development moved directly to `1.0.0-SNAPSHOT`. Historically every minor release carried a
non-empty list of breaking changes, and deprecations were removed aggressively. A 1.0 release should
make SemVer meaningful and reduce unannounced SPI breakage, but that should be confirmed after 1.0 final
ships rather than assumed. The cost is front-loaded: the 1.0 consolidation is the largest breaking
change in the project's history.

### 12.2 Tractus-X and its lag behind upstream

| Tractus-X release | Date | Upstream EDC pinned | Pin released | Pin age at release | Latest upstream then | Minors behind |
|---|---|---|---|---|---|---|
| 0.10.0 | 4 Jun 2025 | 0.13.0 | 21 May 2025 | 14 days | v0.13.0 | 0 |
| 0.11.0 | 1 Oct 2025 | 0.14.0 | 19 Aug 2025 | 43 days | v0.14.1 | 0 |
| 0.12.0 | 9 Mar 2026 | 0.15.1 | 17 Dec 2025 | 82 days | v0.16.0 | 1 |
| 0.12.1 | 21 May 2026 | 0.15.1 | 17 Dec 2025 | 155 days | v0.17.0 | 2 |
| 0.12.2 | 25 Aug 2026 | 0.15.1 | 17 Dec 2025 | 251 days | v0.18.0 | 3 |
| **0.13.0** | **27 Aug 2026** | **0.17.0** | **20 Apr 2026** | **129 days** | **v0.18.0** | **1** |

Three patterns stand out:

1. **Feature releases ship 0–1 minor versions behind upstream**, but the pin age at release has grown
   every cycle: 14, 43, 82 and 129 days.
2. **Patch releases never move the pin**, so the gap widens over a release line's life — 0.12.2 shipped
   on an upstream version 251 days old.
3. **Releases can stall on ecosystem coordination.** 0.13.0 went through four release candidates
   (15 April, 22 May, 22 June and 25 August 2026) before its final release on 27 August; the last
   blocker was a compatibility-test issue between IdentityHub and DCP 0.8.

As of 10 October 2026 the 0.17.0 pin is 173 days old, upstream's latest stable is v0.18.1, and
upstream is in 1.0 release candidates. Tractus-X's 117 modules compile against fine-grained SPI and
library coordinates that 1.0 deletes, and no Tractus-X branch or commit shows 1.0 adoption has begun.
Moving to 1.0 will mean rewriting build files across the repository, and probably reworking the data
plane — likely the most disruptive upstream upgrade Tractus-X has undertaken.

## 13. Extensibility

Both projects use the same mechanism: `ServiceExtension` classes with `@Extension`, `@Provider` and
`@Inject`, discovered through Java's `ServiceLoader` (`META-INF/services`). Skills transfer completely.

**Extending upstream.** Create a Gradle module, depend on the relevant SPI artefacts, implement
`ServiceExtension`, and add the module to your launcher. Total freedom, with two obligations: keep up
with SPI change every two to three months, and on the move to 1.0, rewrite every SPI and library
dependency declaration for the consolidated coordinates. The code inside extensions is largely
unaffected; the build files are not.

**Extending Tractus-X.** Either build a custom runtime that depends on `edc-controlplane-base` and adds
your module — clean, but you now own an image build and a merge burden — or contribute the extension
to Tractus-X so it is maintained, tested and released with the distribution. The second is the
intended path for anything Catena-X-relevant.

**Building your own distribution on upstream, using Tractus-X as a blueprint.**
`edc-controlplane-base/build.gradle.kts`, the Helm charts, the Flyway migration modules and the
compatibility and upgradeability workflows are Apache-2.0 and directly reusable. For a non-Catena-X
dataspace this usually beats both alternatives: upstream's pace and flavours, without reinventing the
operational scaffolding or inheriting automotive-specific semantics.

## 14. Risk register

Severity is the author's judgement: **High** — could invalidate the choice or require replanning;
**Medium** — material cost to absorb; **Low** — manageable with ordinary care.

| Risk | Applies to | Severity | Evidence and mitigation |
|---|---|---|---|
| 1.0 consolidation deletes Maven coordinates | Upstream adopters; Tractus-X later | High | 45→3 SPI and 44→5 library modules. Treat the move to 1.0 as a project with its own budget. New projects should start on 1.0 |
| Tractus-X adoption of EDC 1.0 is slow and disruptive | Tractus-X | High | Pin age rising each cycle; no adoption work visible; classic-flavour architecture. Do not plan on 1.0 features through Tractus-X in the near term |
| Concentrated maintainer base | Both; upstream more | Medium | Top three authors: 93% (upstream), 71% (Tractus-X) of human commits. Track contributor health; budget to contribute fixes yourself |
| Data plane framework removed in 1.0 | Upstream 1.0; Tractus-X later | Medium | No data plane BOMs from RC1. Plan migration of custom `DataSource`/`DataSink` code to the external SDK and DPS |
| EDR handling diverging | Tractus-X consumers | Medium | Upstream moves EDRs into the data plane; Tractus-X relies on `/v3/edrs` and token refresh. Expect consumer-side rework when Tractus-X follows upstream |
| Unsettled management API surface | Tractus-X | Medium | v3, v4 and deprecated v4alpha served together; inherited parts marked deprecated; strategy undecided. Hide version paths behind a client layer |
| 1.0 module layout still changing after RC4 | Upstream | Medium | `jsonld-lib` merged into `core-lib` on `main` after RC4. Pin to 1.0 final, not an RC |
| No automatic PostgreSQL upgrade path in 0.13.0 | Tractus-X | Medium | Chart moved to PostgreSQL 18 on CloudPirates images; a manual database migration is required |
| No versioned schema migrations | Upstream | Medium | `edc.sql.schema.autocreate` only creates tables on an empty database. Build or adopt a migration mechanism (Tractus-X's Flyway modules are a template) |
| Silent misconfiguration on 0.13.0 upgrade | Tractus-X | Medium | `participant.id` now carries the DID; the BPNL moved to `participant.bpnl`. IATP→DCP and DIM→DIV renames affect Helm values, settings and packages |
| Federated catalog data dropped on upgrade | Tractus-X 0.13.0 | Low | Set the opt-out flag before upgrading if the data matters |
| Infrastructure lock-in | Tractus-X | Low | PostgreSQL and HashiCorp Vault are fixed; Vault 1.18.1 is known incompatible |
| No shipped runnable artefact | Upstream | Low | You own image, chart, configuration and upgrade path. Budget for it (Section 15.3) |

## 15. Decision guide

### 15.1 The fast path

```
Joining Catena-X, or is CX-0018 / CX-0152 compliance mandated?
│
├── YES ─► Tractus-X EDC. It is the conformance reference.
│          Alternatives mean reimplementing BPN validation, BDRS,
│          DCP scopes and 28 CX-0152 policy operands.
│
└── NO
    │
    ├── Must you interoperate with connectors still on DSP 0.8?
    │       YES ─► Tractus-X EDC (the only maintained DSP 0.8 binding)
    │
    ├── Do you need multi-tenancy, CEL policies, partner groups,
    │   Management API v5 or multiple dataspace profiles?
    │       YES ─► Upstream EDC 1.0, virtual flavour
    │
    └── A single-tenant connector for a non-Catena-X dataspace?
            └── Can a platform team own the image, chart,
                migrations and upgrades?
                    YES ─► Upstream EDC 1.0, classic flavour,
                           with Tractus-X as the packaging blueprint
                    NO  ─► Tractus-X only as a short-term bridge.
                           You will be maintaining a fork with the
                           Catena-X extensions removed.
```

### 15.2 Scenarios

| Scenario | Choice | Reasoning |
|---|---|---|
| Automotive supplier joining Catena-X | Tractus-X | Conformance is the requirement, not a preference |
| Proof of concept to learn EDC concepts | Tractus-X `tractusx-connector-memory` chart | Fastest route to a working two-connector exchange |
| New dataspace in another sector (health, energy, public sector) | Upstream 1.0 classic, own distribution | Catena-X policy semantics and BPN identity would be wrong for the domain |
| Connector-as-a-service for many participants | Upstream 1.0 virtual | Participant contexts, v5 API with scopes, CEL and partners exist only there |
| Commercial product built on EDC | Upstream 1.0 | Newest SPI, no downstream lag, no Catena-X coupling |
| Existing Catena-X deployment wanting an upstream 1.0 feature | Tractus-X, and wait | Most new upstream features are virtual-flavour and require more than a version bump; a private fork inherits the 1.0 migration |
| Must interoperate with DSP 0.8 partners | Tractus-X | Only maintained DSP 0.8 binding |
| Organisation with an OSS-compliance gate | Tractus-X | Eclipse IP process, dependency file and supply-chain scanning already in place |
| Policy forbids HashiCorp Vault | Upstream, or a Tractus-X fork | Tractus-X images are Vault-only |

### 15.3 Cost of each path

**Upstream** — budget for: a launcher module and dependency selection; a container image; a Helm chart;
a schema migration mechanism for about 20 SQL stores; configuration and secret wiring; your own policy
constraint functions or CEL expressions; an e2e and cross-version upgrade test harness; a data plane
built on the external SDK; and a regular upgrade rhythm. *Estimate, author's judgement:* three to six
engineer-months to reach the operational maturity Tractus-X provides on day one, plus ongoing
maintenance. Start on the 1.0 line to avoid paying the module-consolidation migration later.

**Tractus-X** — accept: Catena-X policy and identity semantics built in; PostgreSQL and HashiCorp Vault;
an upstream base (EDC 0.17.0) two minor versions and a major transition behind; no multi-tenancy, CEL,
Management API v5 or partner groups; an unsettled management-API surface; and migration guides that
must be read line by line — 0.12 to 0.13 alone changes `participant.id` semantics, renames IATP→DCP and
DIM→DIV, drops the federated catalog and requires a manual PostgreSQL migration.

## 16. Recommendation

**In Catena-X, use Tractus-X EDC. Outside Catena-X, use upstream EDC on the 1.0 line, packaged the way
Tractus-X packages it.**

For upstream adopters, pick the flavour deliberately. The classic connector is the closer analogue to
Tractus-X and the simpler deployment. The virtual connector is where upstream's newest capabilities —
CEL, partner groups, v5 and multi-profile support — are landing, at the cost of a more complex security
and tenancy model. Wait for 1.0 final before committing a production build; the module layout is still
moving after RC4.

Whichever you choose, make four architectural commitments deliberately:

1. **Policy vocabulary.** It is the least portable part of a connector deployment. CX-0152 exists only
   in Tractus-X; upstream's CEL and partner groups cover generic cases but only in the virtual flavour.
2. **Data plane strategy.** Upstream 1.0 removes the in-repository data plane framework in favour of an
   external SDK integrated through Data Plane Signaling. Custom `DataSource`/`DataSink` code is the code
   most likely to break in the next 12 months.
3. **EDR handling on the consumer side.** Upstream is moving EDR management into the data plane;
   Tractus-X keeps a control-plane EDR API. Design consumer applications so this can change.
4. **Management API version.** Upstream retires v3 in 1.0; Tractus-X serves v3, v4 and v4alpha with no
   settled strategy. Put version paths behind a client abstraction.

## Appendix A — Verification commands

```bash
# Which upstream EDC version does Tractus-X pin?
curl -s https://raw.githubusercontent.com/eclipse-tractusx/tractusx-edc/main/gradle/libs.versions.toml \
  | grep '^edc ='

# What does the Tractus-X distribution compose and exclude?
curl -s https://raw.githubusercontent.com/eclipse-tractusx/tractusx-edc/main/edc-controlplane/edc-controlplane-base/build.gradle.kts

# Which release line is upstream on, and how far has SPI consolidation gone?
curl -s https://raw.githubusercontent.com/eclipse-edc/Connector/main/gradle.properties | grep '^version'
curl -s https://raw.githubusercontent.com/eclipse-edc/Connector/main/settings.gradle.kts | grep ':spi:'

# Module, SPI and BOM counts at a given tag (example: v1.0.0-RC4)
curl -sL https://codeload.github.com/eclipse-edc/Connector/tar.gz/refs/tags/v1.0.0-RC4 | tar xz
grep -c '^include' Connector-1.0.0-RC4/settings.gradle.kts
grep '^include' Connector-1.0.0-RC4/settings.gradle.kts | grep 'dist:bom'

# Management API versions served at a tag
grep -rhoE '@Path\("/v[0-9]+[a-z]*' Connector-1.0.0-RC4/extensions --include=*.java | sort | uniq -c

# Contributor concentration over the last 12 months (bots excluded)
git clone -q --bare --filter=blob:none https://github.com/eclipse-edc/Connector.git edc.git
git --git-dir=edc.git log --since=2025-10-10 --format='%aN' | grep -viE 'bot' | sort | uniq -c | sort -rn | head
```

## Appendix B — References

| Topic | Location |
|---|---|
| Upstream repository | https://github.com/eclipse-edc/Connector |
| Upstream documentation | https://eclipse-edc.github.io |
| Upstream decision records | `Connector/docs/developer/decision-records/` |
| SPI consolidation | `…/decision-records/2026-06-18-spi-module-consolidation/` |
| Library consolidation | `…/decision-records/2026-06-18-lib-module-consolidation/` |
| Multi-profile virtual connector | `…/decision-records/2026-05-11-multi-profile-virtual-connector/` |
| EDR cache deprecation | `…/decision-records/2026-04-09-edr-cache-deprecation/` |
| Partners and partner groups | `…/decision-records/2026-09-22-partners-and-partner-groups/` |
| Data Plane Signaling specification | https://github.com/eclipse-dataplane-signaling/dataplane-signaling |
| Data plane SDK (Java) | https://github.com/eclipse-dataplane-core/dataplane-sdk-java |
| IdentityHub | https://github.com/eclipse-edc/IdentityHub |
| Technology repositories (AWS, Azure) | https://github.com/eclipse-edc/Technology-Aws · https://github.com/eclipse-edc/Technology-Azure |
| Tractus-X repository | https://github.com/eclipse-tractusx/tractusx-edc |
| Tractus-X migration guides | `tractusx-edc/docs/migration/` |
| Tractus-X Management API walkthrough | `tractusx-edc/docs/usage/management-api-walkthrough/` |
| Tractus-X decision records | `tractusx-edc/docs/development/decision-records/` |
| Tractus-X OpenAPI specifications | https://eclipse-tractusx.github.io/tractusx-edc/openapi/ |
| Tractus-X test reports | https://eclipse-tractusx.github.io/tractusx-edc/allure-reports/ |
| BPN-DID Resolution Service | https://github.com/eclipse-tractusx/bpn-did-resolution-service |
| Catena-X standards | CX-0018 (dataspace connectivity), CX-0152 (policy constraints) |

## Appendix C — Glossary

| Term | Meaning |
|---|---|
| BDRS | BPN-DID Resolution Service — Catena-X service mapping business partner numbers to DIDs |
| BOM | Bill of materials — a Gradle/Maven platform artefact that fixes a set of module versions |
| BPN, BPNL | Business Partner Number; BPNL identifies a legal entity in Catena-X |
| CEL | Common Expression Language, used upstream for configurable policy expressions |
| DCP | Decentralized Claims Protocol — how a connector obtains and verifies credentials (formerly IATP) |
| DID | Decentralized Identifier |
| DIV | Decentralized Identity Verification — the SAP-operated wallet service used in Catena-X (formerly DIM) |
| DPF | Data Plane Framework — upstream's in-repository data plane, removed in 1.0 |
| DPS | Data Plane Signaling — protocol between control plane and data plane |
| DSP | Dataspace Protocol — catalog, negotiation and transfer between connectors |
| EDR | Endpoint Data Reference — address and token a consumer uses to pull data |
| SPI | Service Provider Interface — EDC's extension points |
| STS | Secure Token Service |
| TCK | Technology Compatibility Kit — conformance test suite |
| VC, VP | Verifiable Credential, Verifiable Presentation |
| VCDM | W3C Verifiable Credentials Data Model |

## Appendix D — Document control

| Version | Date | Changes |
|---|---|---|
| 1.0 | 30 Aug 2026 | Initial comparison |
| 1.1 | 10 Oct 2026 | Updated for upstream 1.0 release candidates and Tractus-X 0.13.0 |
| 2.0 | 10 Oct 2026 | Full re-verification against release tags and git history. Corrected policy-operand count, upstream CI JDK, migration count, timing of data plane BOM removal, upstream Kafka and schema-handling status, and contributor assessment. Added: classic and virtual flavours, EDR divergence, external runtime dependencies, measured code composition, lag table, severity-rated risks, glossary. Restructured for delivery |

All figures were verified against the stated tags and repository history on 10 October 2026. Both
projects change quickly; re-verify version-specific statements before committing to a decision.
Appendix A lists the commands used.
