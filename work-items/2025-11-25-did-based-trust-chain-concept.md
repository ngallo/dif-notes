# DID-based Trust Chains for Workloads and Agents

This document is explains concepts related to the proposal in [DIF Work Item Proposal (Draft)](2025-11-24-dif-work-item-proposal.md) and is intended to illustrate the core idea of using DID-based trust chains for workloads and agents.

## Paradigm Shift for Authorization in Distributed Transactions

The current approach aims to define a robust authorization flow that supports workloads and AI Agents.  
This work enables an alternative authorization primitive for distributed environments.

**Current Paradigm:**  
> **Tell me who you are — and what you were allowed, requested, or delegated to perform at the moment we minted a bearer token embedding that snapshot of authorization.**

**New Paradigm:**  
> **Show how you got here**
>
Capability continuity is derived from the verified transaction context and participant provenance.  
Governance and risk adjustments MAY be applied within the distributed transaction itself.  
A terminated transaction MUST invalidate all delegated capabilities associated with it.

This shift is grounded in the model of distributed transactions.  
We must design a model capable of supporting distributed transactions at scale and establishing a consistent security model that applies across the public internet and enterprise/internal environments.

In distributed transactions, **asynchronous flows are the norm**, not the exception.  
Messaging systems, event brokers, and streaming transports dominate real workloads.

Workloads and AI Agents are **first-class participants in distributed transactions**.  
The paradigm is based on a core assumption:

1. **Distributed transactions MUST continue to operate across asynchronous boundaries, including message flows where bearer tokens dissolve.**

2. **Transaction rails MUST enable a consistent security model applicable to both public internet and private/enterprise environments.**

3. **A capability token MUST NOT be treated as valid outside the distributed transaction that delegated it.**

> **NOTE:** OAuth, OIDC, SAML and bearer flows aren’t rejected — they simply become a subset of the broader model.

### Example: Delegated Capability

> You do not need to know the driver’s identity to start the engine.  
> You only need proof of delegated capability.

A **car key** represents a **capability token**:  
it enables the holder to perform a specific action without embedding personal identity.

In a distributed transaction model, a capability token is **valid only for the transaction in which it was delegated** (the authorized trip).  
Outside that transaction scope, the token is **inert**.

## Building the Superset Security Model

Reasoning from trivial cases is meaningless.  
Synchronous handoffs (in-memory, single hop, no queues) always validate the legacy model.  
They do not test its limits and therefore cannot demonstrate a superset.

To prove a superset, we must start from the **principal case**:  
the environment where distributed transactions actually operate — **asynchronous messaging**.

Only in asynchronous flows do the fundamental constraints emerge:

- **Identity continuity MUST be cryptographically preserved through provable delegation. Replacing the Subject identity at any hop creates a new principal and MUST be treated as an attack surface.**
- **Bearer tokens cannot preserve holder-authenticity or capability-context across asynchronous boundaries.**  
- **Minting replacements violates subject integrity.**  
- **Capability must be delegated, not replayed.**

The async boundary is not an edge case —  it is the **proof** that the entire snapshot-based model is a subset.

---

### The Fundamental Constraint

A bearer token **cannot preserve its security semantics or holder binding** across asynchronous boundaries.  
Forwarded as payload, it becomes detached from the holder: signature does not reconstitute holder-binding.  
Therefore, bearer-token identity is a **subset** of the superset model.

---

### Identity Rules in Distributed Transactions

1. **If you are not the Subject, you MUST NOT mint or issue a token on behalf of the Subject.**  
2. **Distributed transactions operate as: Holder → Verifier → Holder → Verifier.**

In-memory handoffs preserve continuity (a Verifier MAY become the next Holder).  
Asynchronous boundaries do not: the next Holder has no Subject token.  
Minting a new token at this point **violates Rule 1**.

> **Token-based identity continuity CANNOT be preserved across asynchronous boundaries.**  
> Such flows become discontinuous unless the underlying protocol was designed to tolerate token fragmentation.

---

### Shifting the Model

The Holder does **not** forward the token.  
The Holder delegates capability to the Verifier via an attestation issued by the Trust Plane.

The Verifier MUST prove:

- its own identity continuity, and  
- that the capability was delegated by the previous hop.

We no longer forward the Subject token.  
We forward **proof of delegated capability**.

The delegated Agent (human, machine, or AI) is anchored to its DID.  
Capability metadata is embedded in the trust envelope.

The Agent is not a token holder —  
it is a **verifiable participant in the capability chain**,  
able to act under delegated authority **without credential leakage**.

## Appendix

- **Transaction rails:** the transport and boundary surfaces that carry distributed execution (e.g., APIs, queues, brokers, message buses).
