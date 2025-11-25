
# **DIF Work Item Proposal (Draft)**

> IMPORTANT NOTE: This document is a copy of the original proposal draft, produced from the work by Nicola Gallo and Kim Hamilton Duffy: https://hackmd.io/@kimdhamilton/H13O5JlbZg

**Status**: *Initial draft, based on Nicola’s reference architecture.*

**Data Model Spec**: *DID-based Trust Chains for Workloads and Agents*

**Key Change**: *Distributed transaction participants must propagate identity continuity. Authorization shifts from token possession to causal identity provenance. Trust lives in the identity graph, not in the token — and the graph persists throughout the distributed transaction.*

## **1. Context and Problem Statement**

Modern enterprise systems increasingly involve **autonomous agents**, microservices, and cross-domain workloads. These systems must:

* authenticate workloads and agents,
* propagate trust across multiple hops,
* verify runtime conditions,
* carry trust across asynchronous boundaries (brokers, queues, streams - e.g. Apache Kafka),
* and operate across domains without central authorities.

Today’s dominant approaches—OAuth/OIDC, mTLS, SPIFFE/SPIRE—were not designed for:

* multi-hop trust propagation,
* portable runtime attestations,
* DID-based identities,
* async-safe trust representations, or
* unified identity models for agents + workloads.


---

## **2. What Exists Today — and What’s Missing**

### **Existing components**

* **OAuth/OIDC / JWTs**
  Great for user authorization, but not designed for hop-level trust, signature chaining, or identity continuity across async boundaries.

* **SPIFFE/SPIRE**
  Strong workload identity within a trust domain, but not DID-compatible and not designed for identity chain propagation.

* **mTLS, VPNs, shared CAs**
  Provide transport security, but no portable trust layer or federated domain model.

* **DIDs, DIDComm, and VCs**
  Provide decentralized identifiers, message authenticity, and attestations, but do not define a profile for multi-hop trust propagation between workloads and agents.

### **Missing pieces**

* A distributed transaction model where **DIDs** represent the **participants**.
* A **DIDComm trust envelope profile** for hop-to-hop propagation (self, peer, attestation).
* A **hash-linked message profile** to build a **trust chain** across hops, where each message can be accepted based on attestation inherited from the verifiable chain, eliminating the need to re-transport the initial attestation and enabling derived authorization from the initial attestation context.
* A standard **attestation wrapper model** for safe ephemeral HTTP/gRPC sessions and unsafe/persisted async environments.
* A **DID-bound claim format** for async flows that preserves trust without signatures.
* A simple **federation bundle** (did.json) for cross-domain trust anchors.
* A **unified identity chain** pattern for workloads and AI agents.

Enterprises are currently inventing bespoke solutions. There is no shared, interoperable standard for **DID-based trust envelopes** suitable for synchronous and asynchronous systems.

---

## **3. Proposed Standards (Brief)**

We propose a data model for enabling DID-based trust chains across workloads and agents.

In distributed transactions, participants must propagate identity continuity, moving the focus of trust from the token to the verifiable chain.

### Identified DataModel components

* **Trust Plane (TP)**: 
Each security boundary exposes a Trust Plane responsible for maintaining the integrity and verifiability of the trust chain. The Trust Plane is identified and anchored via a DID, and Trust Chain signing keys SHOULD be listed under assertionMethod in the DID Document.

* **DIDComm Trust Envelope (TDE)**: 
    A DIDComm v2 message profile for hop-to-hop propagation.
    A TDE MUST include:
    - **Self Identity** — the workload or agent performing the action (DID).
    - **Peer Identity** — the caller or upstream actor (DID).
    - **Attestation reference** — a VC, VP, JWT, SD-JWT, or equivalent.
    - **Chain reference** — an immutable hash reference to the previous hop.
    
    > The Trust Plane MAY wrap, hash, or append a signed hop descriptor to the message. The attestation itself MUST NOT be modified.
    
    > The attestation MAY be embedded for compatibility with existing industrial authorization flows (e.g., JWT). Where possible, it SHOULD be referenced via URI, DID URL, or cryptographic hash to avoid credential propagation across distributed hops.


* **Trust Plane Federation (TPF)**:
Federation happens through DID resolution and signed presentations.
    - Each Trust Plane exposes a DID Document.
    - Federation material consists of:
        - root DIDs or federation trust roots
        - allowed domains
        - verification methods
    - Cross-boundary validation is anchored on DID + VP signed using Trust Plane keys.
    

> This specification defines a Data Model. It does not define the implementation or behavior of any Trust Plane service.
> 
> A Trust Plane is represented by a DID, and its DID Document MUST declare dedicated verification methods (e.g., under assertionMethod) used to sign or transform Trust Chain contributions.
> 
> Cross-domain interoperability is enabled by a DID-Federation bundle that links Trust Plane DIDs and their trust anchors.

**Note on Trust Policies:** *The Trust Plane Federation MAY later be associated with Trust Policies, but these are out of scope for this Data Model specification.
A future specification MAY define how Trust Policies are referenced, validated, and associated to a Trust Plane DID or to a DID Document section.*

> At this stage, Trust Policy References modeling is intentionally deferred.

### **A. DIDComm Trust Envelope (DTE)**

A DIDComm v2 application profile for chaining trust propagation and identity continuity across hops, moving from token possession to causal identity provenance.

> **EXAMPLE DISCLAIMER**
The JSON structures are semantically illustrative, not finalized.
They may contain errors.
The role of this work item is to define the correct data model, not to perfect the current examples.

```jsonc
{
  "from": "did:web:example.com:svcA",          // TP / Self sender DID
  "to": ["did:web:caller.com:svcB"],           // Peer / upstream DID(s)
  "body": {
    "tde": {
        "trustplane": "did:web:trustplane.example.com", // The Trust Plane
        "self": "did:web:example.com:svcA",        // workload or agent executing this hop
        "peer": "did:web:caller.com:svcB",         // caller / upstream participant
        "attestation_ref": "did:web:idp.example.com#vc-1234", 
        // attestation_ref MAY be:
        // - a DID URL (VC/VP reference),
        // - a URI,
        // - a cryptographic hash,
        // - or, where legacy compatibility is required, an embedded JWT / SD-JWT.
        "hop_id": "9e0d9f1b-1c0a-4a3b-9e5b-123456789abc" // optional hop identifier
    },
    "prev_hash": "hl:abc123..."             // hash link to previous hop/envelope
  }
}
```

Each hop **validates** the incoming TDE and derives the current trust state.

The **Self identity authenticates at the boundary Trust Plane**, proving it is the authorized verifier for the current hop as delegated by the upstream holder.

The Trust Plane then issues a new TDE for the next hop (wrapping, hashing, or appending a signed hop descriptor), extending the verifiable trust chain without re-transporting the original attestation.

#### Async DTE

Async transports may carry signatures as opaque bytes, but they do not preserve the verification semantics, ownership guarantees, or replay protections of the envelope.

Message brokers, queues, and event streaming systems (Kafka, NATS, Pulsar, SQS) neither preserve DID ownership guarantees nor maintain the verification semantics of signed envelopes across hops.

The Async-DTE does not carry embedded credentials or portable attestations.
It retains only DID-anchored, hash-derived continuity material.
The Trust Plane verifiably derives a trust state and emits a minimal immutable claim set suitable for asynchronous propagation.

The Async-TDE therefore:

- Does not carry tokens or presentations,
- Does not propagate original credential signatures as authorization context,
- Does not extend the original credential presentation chain,
- Preserves identity continuity via hash linkage.

Original signatures MAY still exist as opaque payloads, but MUST NOT be revalidated, replayed, or reinterpreted as fresh authorization or delegation context.

This supports interoperability with legacy infrastructures while preventing credential leakage.

```jsonc
{
  "from": "did:web:example.com:svcA",
  "to": ["did:web:caller.com:svcB"],
  "body": {
    "tde": {
        "trustplane": "did:web:trustplane.example.com",
        "self": "did:web:example.com:svcA",
        "peer": "did:web:caller.com:svcB",
        "attestation_hash": "hl:def456",
        "hop_id": "urn:uuid:9e0d9f1b-1c0a-4a3b-9e5b-123456789abc"
    },
    "prev_hash": "hl:abc123..."
  }
}
```

The Async-TDE is:

- durable — survives async buffering and retries,
- DID-bound — anchored to workload identity and Trust Plane DID,
- verifiably derived — backed by a previous TDE validation,
- safe to propagate — suitable as Kafka metadata (or equivalent).

The identity graph remains continuous across sync and async flows, but the credential graph does not.

Sync flows preserve verifiable presentation semantics; async flows preserve only hash-derived continuity.

---

### **C. DID Federation Bundle**

A simple JSON document that lists domain trust anchors:

```json
{
  "trustplane": "did:web:trustplane.example.com",
  "domains": {
    "example.com": "did:web:example.com",
    "partner.org": "did:web:partner.org"
  }
}
```

This enables decentralized cross-domain validation without shared CAs.

---

## **4. Example Reference Architecture**

```
        ┌─────────────────────────────┐
        │  Enterprise A Boundary      │
        │  (Trust Chain)              │
        └───────────┬─────────────────┘
                    │
          1. Ingest OAuth / SPIFFE / JWT
          2. Convert to TDE
                    │
                    ▼
         ┌─────────────────────┐
         │  Internal Services  │
         │  (sync hops)        │
         └──────┬───────┬──────┘
                │       │
        ┌───────▼───┐   │  Each hop appends
        │   TDE#1   │   │  to the chain
        └───────────┘   │
                │       │
        ┌───────▼───┐   │
        │   TDE#2   │◄──┘
        └───────────┘
                │
                ▼
     Async boundary (Kafka, queue)
                │
                ▼
        ┌──────────────┐
        │   Kafka msg   │
        │ (with TDE) │
        └──────────────┘
                │
                ▼
    Enterprise B boundary  
       validates using  
      DID Federation Bundle  
                ▼
          issues new TDE  
          and continues chain
```

AI agents participate exactly like workloads:
each with a DID, optional attestations, and presence in TDE/TCLAIM.

## Appendix

### Change of Paradigm

> Centralized protocols think from the point of view of the IdP.
DID Trust Chains think from the point of view of the transaction.

They represent opposite paradigms.

>This is not a new IdP variant.
It is a change of ontology.

When we think in terms of distributed transactions, we get:

- trust = causal identity + verifiable provenance
- cryptographic history of the flow → a chain of custody
- workload/agent identity persists within the graph
- each hop appends hash-linked evidence
- no tokens required → the chain itself becomes the authorization artifact


### Distributed Transactions & Identities

Two fundamental rules for identities in the context of distributed transactions:

1) If you are not the Subject, you must not mint or issue a token on behalf of the Subject.
2) In a distributed transaction you get: **Holder → Verifier | Holder → Verifier**

If every hop is in-memory, the Verifier can become the next         Holder (token exchange works).

But the moment you hit an async boundary (message, queue, broker), the token cannot travel and you are in Rule 2

The new Holder has no Subject token. Minting a new one would violate Rule 1.

> Consequently, token-based identity continuity cannot be preserved across asynchronous boundaries. In such conditions, the interaction flow becomes discontinuous unless the underlying protocol was already designed to tolerate token fragmentation.

Now shift the focus away from tokens and toward capability + identity proof:
	•	The Holder does not pass the token.
	•	The Holder issues a capability attestation to the Verifier: “You are allowed to act on my behalf for this action.”. You need an issuer for this: the Trust Plane.

The Verifier then proves two things:
- It is itself (identity continuity)It was granted that capability by the previous hop
- We are no longer forwarding the original Subject token. We are forwarding proof of delegated capability.

After capability is delegated, anchor the AI Agent to its DID.
Include its capability metadata in the trust envelope.

Now the Agent is not “holding a token”, it is a verifiable actor in the identity chain, capable of acting under delegated authority without credential leakage.

### Trusted AI Agents

AI agents MAY be represented as DIDs and treated as first-class transaction participants.

Each action performed by an agent becomes a continuation of the existing trust chain:

> AI agents have DIDs and are part of the transaction graph.
Trust is causal and cryptographically verifiable across hops.

Instead of carrying tokens, agents inherit verifiable provenance from the previous hop. This preserves identity continuity, intent, and delegated capability across sync/async boundaries — without credential leakage.

We already have the full data model; only the distributed transaction layer is missing. DID can become the backbone of AI agents.

Open issues remain around prompt provenance, agent registries, and lifecycle governance.

### Use Case

A simple sync flow example

- Alice sends a request to an API in RedCorporate (the request includes the JWT access token).
- The API in RedCorporate needs to call an API in GreenCorporate (the RedCorporate API is acting as both Verifier and Holder for the next call).
- This typically requires prior federation and a token exchange.

```text
Alice → JWT → RedCorporate API (Holder/Verifier) → Exchanged JWT → GreenCorporate API (Verifier)
```

This works only while the flow stays synchronous and the two corporations were federated in advance. It does not scale well across the public internet.

Let’s look at the same case when an async flow is required:

- Alice sends a request to an API in RedCorporate.
- The API needs to send a message, but for security it must strip the JWT signature (the RedCorporate API acting as Verifier and Message Publisher).
- The Message Consumer in RedCorporate later needs to call the API in GreenCorporate but at this point, it has no attestation to present. Essentially, it is holding nothing except its own workload identity. 
- The common workaround is to perform a new federation with a fresh identity (service account or workload), not Alice’s.
- The flow becomes a different identity chain.

```text
Alice → JWT → RedCorporate API (Verifier) → Kafka → RedCorporate Consumer (Holder) → GreenCorporate API (Verifier)
```

Essentially, when composing multiple synchronous calls where the receiver can also act as the holder for the next call (via token exchange), the model generally works. When the verifier and the holder are separate components within a distributed transaction hop, the trust continuity collapses.

> Here, the limitations of token/presentation models become evident: they are effective for initiating transactions, but they do not maintain identity continuity across distributed hops by design. This highlights the need for a decentralized trust model that can carry the identity chain throughout the distributed transaction.

Why this matters? So far this behavior was not a major concern on the public internet, and enterprises worked around it using federation, certificates, red/allow lists, etc. AI Agents make this limitation explicit and impossible to ignore.

Enterprises are now inventing bespoke solutions and start storing credentials everywhere to cope with multi-transport architectures and failure handling. This exposes the long-term risk of an “Internet of Shared Credentials,” where trust is transported through copied or stored tokens rather than portable attestations.

Today, there is still no shared, interoperable mechanism that enables DID-based trust propagation across both synchronous and asynchronous systems.

> AI agents are distributed transaction participants with autonomy, but token- or verifier-based permission models cannot propagate the identity chain or the initiating identity’s intent across distributed hops. This is why distributed transactions should propagate the identity chain and be designed for distributed DID participants.