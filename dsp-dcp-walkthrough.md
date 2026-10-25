# One Exchange, End to End: DSP and DCP Together

As of 10 October 2026 · DSP 2025-1, DCP v1.0.1, tractusx-edc 0.13.0, tractusx-identityhub 0.4.1

## 1. What this walkthrough shows

The [DSP guide](dataspace-protocol-guide.md) and the [DCP guide](decentralized-claims-protocol-guide.md) explain each protocol on its own. This walkthrough follows **one** data exchange from start to finish and shows every message both protocols send along the way: a consumer finds a provider's connector, reads its catalog, negotiates a contract, starts a transfer and reads the data. Each step shows the HTTP request, the message body, what the receiver checks, and why.

How to read it:

- **Values.** All identifiers, hosts and tokens are made up but consistent: the offer id in the catalog is the one the consumer requests, the agreement id is the one the transfer uses, and so on. Follow any id from message to message and it lines up.
- **Validated messages.** Every DSP and DCP message body was checked against the official JSON Schemas (DSP tag `2025-1-err2`, DCP tag `v1.0.1`).
- **Shortened tokens.** Tokens are shortened to `eyJ...`, and decoded tokens are shown separately.
- **Implementation behaviour.** What EDC and Tractus-X do at each step was checked against their source code at the versions above.

Read the "one page" sections of both guides first (section 2 in each); this walkthrough assumes you know what a state machine, a self-issued token and a scope are.

## 2. The cast

| | Provider | Consumer |
| --- | --- | --- |
| Company | Acme Batteries, a supplier | Example Motors, an OEM |
| BPNL | `BPNL00000000ACME` | `BPNL000000000OEM` |
| DID | `did:web:ih.provider.example.com` | `did:web:ih.consumer.example.com` |
| DID document | `https://ih.provider.example.com/.well-known/did.json` | `https://ih.consumer.example.com/.well-known/did.json` |
| Credential Service | `https://ih.provider.example.com/api/credentials/v1/participants/provider` | `https://ih.consumer.example.com/api/credentials/v1/participants/consumer` |
| STS (internal) | `https://ih.provider.example.com/api/sts/token` | `https://ih.consumer.example.com/api/sts/token` |
| Connector DSP base | `https://edc.provider.example.com/api/v1/dsp` | `https://edc.consumer.example.com/api/v1/dsp` |
| Data plane public API | `https://dataplane.provider.example.com/api/public` | — |

Both companies run Tractus-X EDC with their own IdentityHub, and both hold credentials from the issuer `did:web:issuer.example.com`, which each side has configured as a trusted issuer.

The identifiers that travel through the exchange:

| What | Value | Created by |
| --- | --- | --- |
| Dataset (EDC asset) | `battery-pass-0001` | Provider, when registering the asset |
| Offer | `YmF0dGVyeS1wYXNzLW9mZmVy:YmF0dGVyeS1wYXNzLTAwMDE=:N2YxYzJhOGUtMGI1ZC00YzNlLTlhNmYtMmQ0YjhlMWM3YTkw` | Provider's EDC, per catalog request. The three base64 parts decode to the contract definition `battery-pass-offer`, the asset `battery-pass-0001` and a random UUID |
| Negotiation, consumer side | `urn:uuid:3b8f1d2e-6c4a-4e9b-8f7d-0a1b2c3d4e01` (`consumerPid`) | Consumer |
| Negotiation, provider side | `urn:uuid:9e7d5c3b-1a2f-4b6e-8d0c-5f4e3d2c1b02` (`providerPid`) | Provider |
| Agreement | `urn:uuid:5d2c8a71-3f4e-4b9a-a6c2-7e1f0d9b8c03` | Provider |
| Transfer, consumer side | `urn:uuid:1f2e3d4c-5b6a-4798-8a9b-0c1d2e3f4a04` (`consumerPid`) | Consumer |
| Transfer, provider side | `urn:uuid:8a9b0c1d-2e3f-4a5b-9c6d-7e8f9a0b1c05` (`providerPid`) | Provider |

## 3. What must be in place first

Nothing in this walkthrough works unless the following is already true. Most "it doesn't work" problems turn out to be one of these.

| Who | What | Why it matters |
| --- | --- | --- |
| Both | DID document published at the HTTPS address the DID resolves to, with a signing key and a `CredentialService` entry | The other side resolves it on every request |
| Both | `MembershipCredential`, `BpnCredential` and `DataExchangeGovernanceCredential` stored in their IdentityHub | Tractus-X asks for all three on every DSP 2025-1 request, in both directions |
| Both | The issuer's DID in `dcp.trustedIssuers` | Credentials from any other issuer are rejected |
| Both | STS client credentials configured in the connector (`dcp.sts.oauth.*`) | Without them the connector cannot sign a single request |
| Provider | A `DataService` entry in its DID document pointing at its version endpoint (`dcp.didService.selfRegistration.enabled`) | Lets consumers find the connector from the DID alone |
| Provider | Asset `battery-pass-0001`, two policies and the contract definition `battery-pass-offer` binding them | Without a contract definition the asset never appears in any catalog |

The provider's two policies, which come back at several points below:

- **Access policy:** `Membership eq active`. It decides who may see the offer. It is evaluated by the provider only and never sent over DSP.
- **Contract policy:** `FrameworkAgreement eq DataExchangeGovernance:1.0` and `UsagePurpose isAnyOf cx.circular.dpp:1`. It becomes the offer in the catalog and later the agreement. CX-0152 also requires every policy to declare the Catena-X `profile` (`https://w3id.org/catenax/2025/9/policy/`). Like the CX-0152 samples and the Tractus-X walkthrough, the examples here leave it out (primer section 11).

## 4. Step 1: find the provider's connector

The consumer knows only the provider's BPNL, or its DID. The BPN-DID Resolution Service (BDRS) turns a BPNL into a DID (primer sections 9 and 12). From the DID, the consumer finds the connector and the DSP versions it speaks. Tractus-X's connector discovery API (DSP guide section 10) runs these two calls for you.

**1a. Resolve the provider's DID.** `did:web:ih.provider.example.com` resolves to `https://ih.provider.example.com/.well-known/did.json`. The parts that matter here:

```json
{
  "@context": ["https://www.w3.org/ns/did/v1"],
  "id": "did:web:ih.provider.example.com",
  "service": [
    {
      "id": "did:web:ih.provider.example.com#credential-service",
      "type": "CredentialService",
      "serviceEndpoint": "https://ih.provider.example.com/api/credentials/v1/participants/provider"
    },
    {
      "id": "did:web:ih.provider.example.com#dsp",
      "type": "DataService",
      "serviceEndpoint": "https://edc.provider.example.com/api/v1/dsp/.well-known/dspace-version"
    }
  ]
}
```

**1b. Ask the connector which DSP versions it speaks.** The version endpoint is unauthenticated, so no token is needed yet.

```http
GET https://edc.provider.example.com/api/v1/dsp/.well-known/dspace-version
```

```json
{
  "protocolVersions": [
    { "version": "2025-1", "path": "/2025-1", "binding": "HTTPS" },
    { "version": "v0.8", "path": "/", "binding": "HTTPS" }
  ]
}
```

`path` is appended to the address the version endpoint lives under, so DSP 2025-1 is at `https://edc.provider.example.com/api/v1/dsp/2025-1`. That is the `<base>` for every DSP call below. In the Management API the consumer will use `counterPartyAddress` = that URL, `protocol` = `dataspace-protocol-http:2025-1` and `counterPartyId` = the provider's DID.

## 5. Step 2: read the catalog

This is the first call that crosses between the companies, and DCP runs in full before the provider answers.

```mermaid
sequenceDiagram
    autonumber
    participant CE as Consumer EDC
    participant CIH as Consumer IdentityHub
    participant PE as Provider EDC
    participant PIH as Provider IdentityHub
    CE->>CIH: STS token request (audience provider, three read scopes)
    CIH-->>CE: Self-issued token with access token
    CE->>PE: POST /2025-1/catalog/request, Bearer consumer token
    PE->>CIH: GET /.well-known/did.json (consumer DID)
    Note over PE: Validate consumer token
    PE->>PIH: STS token request (audience consumer, token = consumer access token)
    PIH-->>PE: Provider's own self-issued token
    PE->>CIH: POST .../presentations/query, Bearer provider token
    CIH-->>PE: PresentationResponseMessage
    Note over PE: Validate credentials, evaluate access policy
    PE-->>CE: 200 Catalog
```

### 2.1 The consumer gets a token from its STS

Before sending anything, the consumer's EDC decides which credentials to let the provider read. Over DSP 2025-1 Tractus-X always grants three (DCP guide section 11):

```http
POST https://ih.consumer.example.com/api/sts/token
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
&client_id=consumer
&client_secret=<from the vault>
&audience=did:web:ih.provider.example.com
&bearer_access_scope=org.eclipse.tractusx.vc.type:MembershipCredential:read org.eclipse.tractusx.vc.type:DataExchangeGovernanceCredential:read org.eclipse.tractusx.vc.type:BpnCredential:read
```

The STS returns a self-issued ID token. Decoded, its header and payload are:

```json
{ "alg": "ES256", "typ": "JWT", "kid": "did:web:ih.consumer.example.com#key-1" }
```

```json
{
  "iss": "did:web:ih.consumer.example.com",
  "sub": "did:web:ih.consumer.example.com",
  "aud": "did:web:ih.provider.example.com",
  "jti": "6a0c1e57-2b8d-4f3a-9e61-0d7c5b4a3f21",
  "iat": 1791640800,
  "exp": 1791641100,
  "token": "eyJ...consumer-access-token..."
}
```

### 2.2 The consumer sends the DSP request

```http
POST https://edc.provider.example.com/api/v1/dsp/2025-1/catalog/request
Authorization: Bearer eyJ...consumer-token...
Content-Type: application/json
```

```json
{
  "@context": ["https://w3id.org/dspace/2025/1/context.jsonld"],
  "@type": "CatalogRequestMessage"
}
```

### 2.3 The provider checks who is calling

The provider's EDC resolves `did:web:ih.consumer.example.com`, takes the key named by `kid`, and checks the token (DCP guide section 6):

- `iss` equals `sub`.
- `aud` is its own DID.
- The signature is valid.
- `exp` has not passed.
- The `jti` has not been seen before.

A failure here ends the request with 401.

### 2.4 The provider fetches the consumer's credentials

The provider works out which credentials it needs from the policies it is about to evaluate: `Membership` in the access policy gives a MembershipCredential scope, and the Tractus-X defaults add the other two. It asks its own STS for a token addressed to the consumer, carrying the consumer's access token:

```http
POST https://ih.provider.example.com/api/sts/token
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials&client_id=provider&client_secret=<from the vault>&audience=did:web:ih.consumer.example.com&token=eyJ...consumer-access-token...
```

Then it queries the consumer's Credential Service, found through the `CredentialService` entry in the consumer's DID document:

```http
POST https://ih.consumer.example.com/api/credentials/v1/participants/consumer/presentations/query
Authorization: Bearer eyJ...provider-token...
Content-Type: application/json
```

```json
{
  "@context": ["https://w3id.org/dspace-dcp/v1.0/dcp.jsonld"],
  "type": "PresentationQueryMessage",
  "scope": [
    "org.eclipse.tractusx.vc.type:MembershipCredential:read",
    "org.eclipse.tractusx.vc.type:DataExchangeGovernanceCredential:read",
    "org.eclipse.tractusx.vc.type:BpnCredential:read"
  ]
}
```

The consumer's IdentityHub checks the provider's token, then checks that the embedded access token was issued for this provider and covers these scopes. It returns one presentation, a JWT signed by the consumer, containing the three credentials:

```json
{
  "@context": ["https://w3id.org/dspace-dcp/v1.0/dcp.jsonld"],
  "type": "PresentationResponseMessage",
  "presentation": ["eyJ...consumer-signed-presentation..."]
}
```

The provider validates it (DCP guide section 7):

- The presentation signature is the consumer's.
- Each credential is signed by `did:web:issuer.example.com`, which is trusted, and is within its validity period.
- The issuer's status list says no credential is revoked.
- All three requested types are present.

### 2.5 The provider evaluates the access policy

The verified credentials become the consumer's claims. The access policy `Membership eq active` passes because a MembershipCredential is present. That is all the Catena-X membership function checks: the credential's existence, with its signature, validity and revocation already verified.

### 2.6 The provider answers with the catalog

```json
{
  "@context": [
    "https://w3id.org/dspace/2025/1/context.jsonld",
    "https://w3id.org/catenax/2025/9/policy/context.jsonld"
  ],
  "@id": "urn:uuid:0c7e2f4a-9b1d-4e8c-a5f3-6d2b1e0a9c77",
  "@type": "Catalog",
  "participantId": "did:web:ih.provider.example.com",
  "service": [
    {
      "@id": "urn:uuid:4aa2dcc8-4d2d-469e-a634-8394a8834d77",
      "@type": "DataService",
      "endpointURL": "https://edc.provider.example.com/api/v1/dsp/2025-1"
    }
  ],
  "dataset": [
    {
      "@id": "battery-pass-0001",
      "@type": "Dataset",
      "dct:type": { "@id": "https://w3id.org/catenax/taxonomy#Submodel" },
      "https://w3id.org/catenax/ontology/common#version": "3.0",
      "https://admin-shell.io/aas/3/0/HasSemantics/semanticId": "urn:samm:io.catenax.battery.battery_pass:6.1.0#BatteryPass",
      "hasPolicy": [
        {
          "@id": "YmF0dGVyeS1wYXNzLW9mZmVy:YmF0dGVyeS1wYXNzLTAwMDE=:N2YxYzJhOGUtMGI1ZC00YzNlLTlhNmYtMmQ0YjhlMWM3YTkw",
          "@type": "Offer",
          "permission": [
            {
              "action": "use",
              "constraint": [
                {
                  "and": [
                    { "leftOperand": "FrameworkAgreement", "operator": "eq", "rightOperand": "DataExchangeGovernance:1.0" },
                    { "leftOperand": "UsagePurpose", "operator": "isAnyOf", "rightOperand": ["cx.circular.dpp:1"] }
                  ]
                }
              ]
            }
          ]
        }
      ],
      "distribution": [
        { "@type": "Distribution", "format": "HttpData-PULL", "accessService": "urn:uuid:4aa2dcc8-4d2d-469e-a634-8394a8834d77" }
      ]
    }
  ]
}
```

What the consumer reads from it:

- The dataset id, and the three properties Catena-X requires or recommends on every dataset:
  - `dct:type` from the Catena-X taxonomy, here `Submodel` (CX-0018 and CX-0002).
  - `cx-common:version`, which for a submodel is the AAS specification version.
  - `semanticId`, the aspect model URN, so consumers can filter catalogs by data model.
- The offer, with its id and rules. These are copied unchanged into the next step.
- The `HttpData-PULL` format, meaning the consumer will pull the data.
- The `endpointURL` where negotiations for this dataset start.

The access policy is not in the catalog; the consumer only sees that it passed. A consumer that failed it would get a catalog with no datasets and no error.

## 6. Step 3: negotiate the contract

```mermaid
sequenceDiagram
    autonumber
    participant CE as Consumer EDC
    participant PE as Provider EDC
    CE->>PE: POST /2025-1/negotiations/request (ContractRequestMessage)
    Note over PE: DCP check of consumer, evaluate contract policy
    PE-->>CE: 201 ContractNegotiation REQUESTED
    PE->>CE: POST /2025-1/negotiations/{consumerPid}/agreement (ContractAgreementMessage)
    Note over CE: DCP check of provider, compare agreement with offer
    CE-->>PE: 200, both AGREED
    CE->>PE: POST /2025-1/negotiations/{providerPid}/agreement/verification
    Note over PE: DCP check of consumer
    PE-->>CE: 200, both VERIFIED
    PE->>CE: POST /2025-1/negotiations/{consumerPid}/events (FINALIZED)
    Note over CE: DCP check of provider
    CE-->>PE: 200, both FINALIZED
```

Every arrow carries a fresh self-issued token from the sender's STS, and every receiver repeats the DCP check from step 2: token, presentation query, credential validation. That includes the provider's callbacks: the consumer checks the provider's credentials just as thoroughly. Each side runs the check against the policy of the latest offer. In Tractus-X both sides always ask for the three default credentials; on the contract request the provider also derives scopes from the policy (`FrameworkAgreement` gives `DataExchangeGovernanceCredential`, already among the defaults). The Catena-X scope extractor only looks at catalog, contract and transfer requests, so the checks on callbacks use the defaults alone.

### 3.1 The consumer requests the offer

The consumer starts this through its Management API (`POST /v3/contractnegotiations`), and its EDC sends:

```http
POST https://edc.provider.example.com/api/v1/dsp/2025-1/negotiations/request
Authorization: Bearer eyJ...consumer-token...
Content-Type: application/json
```

```json
{
  "@context": [
    "https://w3id.org/dspace/2025/1/context.jsonld",
    "https://w3id.org/catenax/2025/9/policy/context.jsonld"
  ],
  "@type": "ContractRequestMessage",
  "consumerPid": "urn:uuid:3b8f1d2e-6c4a-4e9b-8f7d-0a1b2c3d4e01",
  "offer": {
    "@id": "YmF0dGVyeS1wYXNzLW9mZmVy:YmF0dGVyeS1wYXNzLTAwMDE=:N2YxYzJhOGUtMGI1ZC00YzNlLTlhNmYtMmQ0YjhlMWM3YTkw",
    "@type": "Offer",
    "target": "battery-pass-0001",
    "permission": [
      {
        "action": "use",
        "constraint": [
          {
            "and": [
              { "leftOperand": "FrameworkAgreement", "operator": "eq", "rightOperand": "DataExchangeGovernance:1.0" },
              { "leftOperand": "UsagePurpose", "operator": "isAnyOf", "rightOperand": ["cx.circular.dpp:1"] }
            ]
          }
        ]
      }
    ]
  },
  "callbackAddress": "https://edc.consumer.example.com/api/v1/dsp/2025-1"
}
```

Compared with the catalog entry, only two things changed: `target` was added (an offer inside a message must name its dataset, DSP guide section 7), and the message carries the consumer's own `consumerPid` and `callbackAddress`. The rules are identical.

### 3.2 The provider checks the consumer and the contract policy

The provider runs the DCP check. This time its scopes come from the contract policy too: `FrameworkAgreement` maps to `DataExchangeGovernanceCredential` (DCP guide section 11). Then it evaluates the contract policy against the verified credentials. This is where a credential turns into a yes or no ([source](https://github.com/eclipse-tractusx/tractusx-edc/blob/0.13.0/edc-extensions/cx-policy/src/main/java/org/eclipse/tractusx/edc/policy/cx/framework/FrameworkAgreementConstraintFunction.java)):

| Constraint | What the Catena-X function checks |
| --- | --- |
| `FrameworkAgreement eq DataExchangeGovernance:1.0` | The right operand splits into a subtype and a version. There must be a credential of type `DataExchangeGovernanceCredential` whose `credentialSubject.contractVersion` is `1.0` |
| `UsagePurpose isAnyOf cx.circular.dpp:1` | No credential involved, and the connector does not check the value either: it only insists on the `isAnyOf` operator ([source](https://github.com/eclipse-tractusx/tractusx-edc/blob/0.13.0/edc-extensions/cx-policy/src/main/java/org/eclipse/tractusx/edc/policy/cx/usage/UsagePurposeConstraintFunction.java)). The purpose is a contractual statement the consumer accepts, not a claim anyone proves |
| Membership | CX-0152 requires providers to check membership on every request. Tractus-X requests the MembershipCredential by default, and access policies normally carry the `Membership` constraint, as here |

So the relevant part of the consumer's DataExchangeGovernanceCredential looks like this (other fields omitted):

```json
{
  "type": ["VerifiableCredential", "DataExchangeGovernanceCredential"],
  "issuer": "did:web:issuer.example.com",
  "credentialSubject": {
    "id": "did:web:ih.consumer.example.com",
    "holderIdentifier": "BPNL000000000OEM",
    "group": "UseCaseFramework",
    "useCase": "<the framework's use case name>",
    "contractTemplate": "<URL of the signed framework agreement template>",
    "contractVersion": "1.0"
  }
}
```

All six `credentialSubject` fields are required by the credential's schema in CX-0050; only `contractVersion` decides this constraint.

A constraint such as `BusinessPartnerNumber isAnyOf [...]` would read the `bpn` claim of the BpnCredential instead (primer section 11). Both checks pass, so the provider creates its copy of the negotiation and answers:

```http
HTTP/1.1 201 Created
Content-Type: application/json
```

```json
{
  "@context": ["https://w3id.org/dspace/2025/1/context.jsonld"],
  "@type": "ContractNegotiation",
  "providerPid": "urn:uuid:9e7d5c3b-1a2f-4b6e-8d0c-5f4e3d2c1b02",
  "consumerPid": "urn:uuid:3b8f1d2e-6c4a-4e9b-8f7d-0a1b2c3d4e01",
  "state": "REQUESTED"
}
```

From now on both ids travel in every message.

### 3.3 The provider sends the agreement

EDC providers do not counter-offer, so the provider moves straight to an agreement and calls the consumer back under its `callbackAddress`:

```http
POST https://edc.consumer.example.com/api/v1/dsp/2025-1/negotiations/urn:uuid:3b8f1d2e-6c4a-4e9b-8f7d-0a1b2c3d4e01/agreement
Authorization: Bearer eyJ...provider-token...
Content-Type: application/json
```

```json
{
  "@context": [
    "https://w3id.org/dspace/2025/1/context.jsonld",
    "https://w3id.org/catenax/2025/9/policy/context.jsonld"
  ],
  "@type": "ContractAgreementMessage",
  "providerPid": "urn:uuid:9e7d5c3b-1a2f-4b6e-8d0c-5f4e3d2c1b02",
  "consumerPid": "urn:uuid:3b8f1d2e-6c4a-4e9b-8f7d-0a1b2c3d4e01",
  "agreement": {
    "@id": "urn:uuid:5d2c8a71-3f4e-4b9a-a6c2-7e1f0d9b8c03",
    "@type": "Agreement",
    "target": "battery-pass-0001",
    "timestamp": "2026-10-10T09:15:02Z",
    "assigner": "did:web:ih.provider.example.com",
    "assignee": "did:web:ih.consumer.example.com",
    "permission": [
      {
        "action": "use",
        "constraint": [
          {
            "and": [
              { "leftOperand": "FrameworkAgreement", "operator": "eq", "rightOperand": "DataExchangeGovernance:1.0" },
              { "leftOperand": "UsagePurpose", "operator": "isAnyOf", "rightOperand": ["cx.circular.dpp:1"] }
            ]
          }
        ]
      }
    ]
  }
}
```

Path ids in this walkthrough are written out in full for clarity. The consumer then does three things:

1. **It checks the provider.** It verifies the provider's token and queries the provider's Credential Service for the same three credentials.
2. **It compares the agreement with the offer it sent.** If the rules differ in any way, the consumer terminates the negotiation here.
3. **It records the agreement.** The agreement id `urn:uuid:5d2c8a71-...` is what the transfer will need.

It answers 200, and both sides are AGREED.

### 3.4 The consumer verifies, the provider finalizes

```http
POST https://edc.provider.example.com/api/v1/dsp/2025-1/negotiations/urn:uuid:9e7d5c3b-1a2f-4b6e-8d0c-5f4e3d2c1b02/agreement/verification
Authorization: Bearer eyJ...consumer-token...
Content-Type: application/json
```

```json
{
  "@context": ["https://w3id.org/dspace/2025/1/context.jsonld"],
  "@type": "ContractAgreementVerificationMessage",
  "providerPid": "urn:uuid:9e7d5c3b-1a2f-4b6e-8d0c-5f4e3d2c1b02",
  "consumerPid": "urn:uuid:3b8f1d2e-6c4a-4e9b-8f7d-0a1b2c3d4e01"
}
```

```http
POST https://edc.consumer.example.com/api/v1/dsp/2025-1/negotiations/urn:uuid:3b8f1d2e-6c4a-4e9b-8f7d-0a1b2c3d4e01/events
Authorization: Bearer eyJ...provider-token...
Content-Type: application/json
```

```json
{
  "@context": ["https://w3id.org/dspace/2025/1/context.jsonld"],
  "@type": "ContractNegotiationEventMessage",
  "providerPid": "urn:uuid:9e7d5c3b-1a2f-4b6e-8d0c-5f4e3d2c1b02",
  "consumerPid": "urn:uuid:3b8f1d2e-6c4a-4e9b-8f7d-0a1b2c3d4e01",
  "eventType": "FINALIZED"
}
```

Both sides are now FINALIZED. On the consumer, `GET /v3/contractnegotiations/{id}` shows `FINALIZED` and a `contractAgreementId`; that is the moment to start a transfer.

## 7. Step 4: transfer and read the data

```mermaid
sequenceDiagram
    autonumber
    participant APP as Consumer app
    participant CE as Consumer EDC
    participant PE as Provider EDC
    participant PDP as Provider data plane
    CE->>PE: POST /2025-1/transfers/request (TransferRequestMessage)
    Note over PE: DCP check, agreement policy, prepare data plane
    PE-->>CE: 201 TransferProcess REQUESTED
    PE->>CE: POST /2025-1/transfers/{consumerPid}/start (dataAddress with token)
    CE-->>PE: 200, both STARTED
    APP->>CE: GET /v3/edrs/{transferProcessId}/dataaddress?auto_refresh=true
    CE-->>APP: EDR (endpoint and token)
    APP->>PDP: GET /api/public, Authorization token
    PDP-->>APP: The passport data
```

### 4.1 The consumer requests the transfer

```http
POST https://edc.provider.example.com/api/v1/dsp/2025-1/transfers/request
Authorization: Bearer eyJ...consumer-token...
Content-Type: application/json
```

```json
{
  "@context": ["https://w3id.org/dspace/2025/1/context.jsonld"],
  "@type": "TransferRequestMessage",
  "consumerPid": "urn:uuid:1f2e3d4c-5b6a-4798-8a9b-0c1d2e3f4a04",
  "agreementId": "urn:uuid:5d2c8a71-3f4e-4b9a-a6c2-7e1f0d9b8c03",
  "format": "HttpData-PULL",
  "callbackAddress": "https://edc.consumer.example.com/api/v1/dsp/2025-1"
}
```

A pull request carries no `dataAddress`; the provider will supply one. The provider then:

1. Runs the DCP check, this time against the agreement's policy.
2. Confirms the agreement belongs to this consumer.
3. Asks its data plane to prepare access to the asset's backend and issue an access token, over Data Plane Signaling (primer section 8).
4. Answers 201:

```json
{
  "@context": ["https://w3id.org/dspace/2025/1/context.jsonld"],
  "@type": "TransferProcess",
  "providerPid": "urn:uuid:8a9b0c1d-2e3f-4a5b-9c6d-7e8f9a0b1c05",
  "consumerPid": "urn:uuid:1f2e3d4c-5b6a-4798-8a9b-0c1d2e3f4a04",
  "state": "REQUESTED"
}
```

### 4.2 The provider starts the transfer

```http
POST https://edc.consumer.example.com/api/v1/dsp/2025-1/transfers/urn:uuid:1f2e3d4c-5b6a-4798-8a9b-0c1d2e3f4a04/start
Authorization: Bearer eyJ...provider-token...
Content-Type: application/json
```

```json
{
  "@context": ["https://w3id.org/dspace/2025/1/context.jsonld"],
  "@type": "TransferStartMessage",
  "providerPid": "urn:uuid:8a9b0c1d-2e3f-4a5b-9c6d-7e8f9a0b1c05",
  "consumerPid": "urn:uuid:1f2e3d4c-5b6a-4798-8a9b-0c1d2e3f4a04",
  "dataAddress": {
    "@type": "DataAddress",
    "endpointType": "https://w3id.org/idsa/v4.1/HTTP",
    "endpoint": "https://dataplane.provider.example.com/api/public",
    "endpointProperties": [
      { "@type": "EndpointProperty", "name": "authorization", "value": "eyJ...data-plane-token..." }
    ]
  }
}
```

This data address is the **EDR**. Tractus-X adds token-refresh properties alongside the token: a refresh endpoint, a refresh token, the expiry time and an audience. The consumer checks the provider's identity once more, stores the EDR in its vault and answers 200. Both sides are now STARTED.

### 4.3 The consumer's application reads the data

This part is outside DSP: it is plain HTTP against the provider's data plane. The application first gets the EDR from its own connector (the id in the path is the consumer's transfer process id from the Management API):

```http
GET https://edc.consumer.example.com/management/v3/edrs/{transferProcessId}/dataaddress?auto_refresh=true
X-Api-Key: <consumer management key>
```

```json
{
  "@type": "DataAddress",
  "endpointType": "https://w3id.org/idsa/v4.1/HTTP",
  "endpoint": "https://dataplane.provider.example.com/api/public",
  "authorization": "eyJ...data-plane-token...",
  "tx-auth:refreshEndpoint": "https://dataplane.provider.example.com/api/public/token",
  "tx-auth:refreshToken": "eyJ...refresh-token...",
  "tx-auth:expiresIn": "300",
  "tx-auth:refreshAudience": "did:web:ih.consumer.example.com"
}
```

Then it calls the data plane with the token as-is, with no `Bearer` prefix:

```http
GET https://dataplane.provider.example.com/api/public
Authorization: eyJ...data-plane-token...
```

The provider's data plane checks the token (it issued it), forwards the call to the backend registered on the asset, and streams back the battery passport. If the asset was registered with `proxyPath` and `proxyQueryParams`, extra path segments and query parameters are forwarded too.

The token lasts 300 seconds. `auto_refresh=true` on the EDR call returns a refreshed token when needed; alternatives are `POST /v3/edrs/{id}/refresh` or the consumer-side data plane proxy, which refreshes automatically ([EDR walkthrough](https://github.com/eclipse-tractusx/tractusx-edc/blob/0.13.0/docs/usage/management-api-walkthrough/07_edrs.md)). Never negotiate again just to get a new token.

### 4.4 Ending the transfer

An HTTP pull of an API is non-finite: the transfer stays STARTED, and the token can keep being refreshed, until one side ends it. When the consumer is done it terminates through `POST /v3/transferprocesses/{id}/terminate`, which sends:

```http
POST https://edc.provider.example.com/api/v1/dsp/2025-1/transfers/urn:uuid:8a9b0c1d-2e3f-4a5b-9c6d-7e8f9a0b1c05/termination
Authorization: Bearer eyJ...consumer-token...
Content-Type: application/json
```

```json
{
  "@context": ["https://w3id.org/dspace/2025/1/context.jsonld"],
  "@type": "TransferTerminationMessage",
  "providerPid": "urn:uuid:8a9b0c1d-2e3f-4a5b-9c6d-7e8f9a0b1c05",
  "consumerPid": "urn:uuid:1f2e3d4c-5b6a-4798-8a9b-0c1d2e3f4a04",
  "code": "done",
  "reason": ["Consumer finished reading the passport"]
}
```

The provider revokes the data plane token, and both sides move to TERMINATED. The agreement remains valid, so the consumer can start a new transfer later without negotiating again.

## 8. What each side holds afterwards

| | Provider | Consumer |
| --- | --- | --- |
| Negotiation | `urn:uuid:9e7d...` FINALIZED, linked to the consumer's id | `urn:uuid:3b8f...` FINALIZED, linked to the provider's id |
| Agreement | `urn:uuid:5d2c...`, assigner itself, assignee the consumer | The same agreement, identical rules |
| Transfer | `urn:uuid:8a9b...` TERMINATED | `urn:uuid:1f2e...` TERMINATED |
| Tokens and credentials | Nothing kept from the consumer except, in Tractus-X, its presentations in the 24-hour cache | The EDR in the vault until deleted; the provider's presentations in its cache |

No central system saw any of this: DID documents, credentials and status lists were fetched directly from each party and the issuer.

## 9. When things go wrong

**Retries.** EDC retries each outgoing DSP message with exponential backoff: 7 attempts starting at a 1-second delay by default, set by `edc.negotiation.send.retry.limit`, `edc.negotiation.send.retry.base-delay.ms`, `edc.transfer.send.retry.limit` and `edc.transfer.send.retry.base-delay.ms`. When the last attempt fails, EDC gives up and terminates the process on its side. If even the first contract request never got through, the negotiation goes straight to TERMINATED; later failures go to TERMINATING, and EDC tries to tell the other side.

**No timeouts in the protocol.** DSP has no deadlines. If a callback never arrives, because the consumer's DSP endpoint is unreachable from the internet for example, the consumer's negotiation simply stays in REQUESTED. Code that waits for FINALIZED needs its own timeout.

**Where failures show up** in this walkthrough:

| Step | Symptom | Usual cause |
| --- | --- | --- |
| 1 | No `DataService` entry, or 404 from the version endpoint | Self-registration off; wrong DSP base path |
| 2.1 | Request never sent; STS or OAuth error in the consumer's log | Wrong STS client id or secret alias |
| 2.3–2.4 | 401 from the provider | Wrong `aud` (stale DID in `counterPartyId`), unresolvable DID, clock skew, missing or untrusted credential |
| 2.5 | 200 with an empty catalog | Credentials fine, but the access policy failed |
| 3.2–3.3 | Negotiation TERMINATED | Missing DataExchangeGovernanceCredential, offer edited, or the provider failed the consumer's check of *its* credentials |
| 4.3 | 403 from the data plane | Token expired; refresh it |

The DCP guide's section 15 and the DSP guide's section 12 go deeper on each.

## 10. What the protocols do not guarantee

- **Nothing technical stops misuse after delivery.** Once the bytes reach the consumer, DSP and DCP have done their job. The agreement's usage rules (purpose `cx.circular.dpp:1` here) are a binding contract between the companies, enforced legally, not cryptographically.
- **Ongoing access is re-checked, delivered data is not.** EDC's policy monitor re-evaluates the agreement during long-running transfers and can terminate them (primer section 11). Tractus-X providers can also retire an agreement early, which blocks new transfers under it.
- **Revocation takes a while to bite.** A revoked credential is noticed only when the verifier's cached status list expires (15 minutes by default in EDC); cached presentations are re-checked against that list.
- **Identity is not trustworthiness.** DCP proves the consumer is a Catena-X member that signed the framework agreement. It says nothing about whether the consumer will honour the usage purpose.

## 11. See it yourself

These commands work against any deployment you can reach. Replace the hosts with your own.

Resolve a DID (here, a `did:web` with no path):

```bash
curl -s https://ih.provider.example.com/.well-known/did.json | jq '.service'
```

List the DSP versions a connector speaks:

```bash
curl -s https://edc.provider.example.com/api/v1/dsp/.well-known/dspace-version | jq
```

Decode any JWT you capture (a self-issued token, a presentation, a credential) without sending it anywhere:

```bash
python3 -c 'import sys,json,base64; p=sys.argv[1].split(".")[1]; print(json.dumps(json.loads(base64.urlsafe_b64decode(p+"="*(-len(p)%4))),indent=2))' "$TOKEN"
```

Watch a negotiation from the consumer side, and fetch the EDR once a transfer has started:

```bash
curl -s -H "X-Api-Key: $KEY" "$CONSUMER_MGMT/v3/contractnegotiations/$NEGOTIATION_ID/state" | jq
curl -s -H "X-Api-Key: $KEY" "$CONSUMER_MGMT/v3/edrs/$TRANSFER_ID/dataaddress?auto_refresh=true" | jq
```

To see the DSP and DCP calls themselves, run the exchange against a local deployment such as the Tractus-X umbrella chart (primer section 16) and raise the connector's log level. Avoid jwt.io and similar websites for real tokens: pasting a live token into a third-party site leaks it.

Sources for this document are linked inline in each section and in the two guides. As of 10 October 2026.
