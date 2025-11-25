# DID-based Trust Chains for Workloads and Agents

This document is a simplified concept related to the proposal in [DIF Work Item Proposal (Draft)](2025-11-24-dif-work-item-proposal.md) and is intended to illustrate the core idea of using DID-based trust chains for workloads and agents.

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

2. **Transaction rails MUST define a consistent security model applicable to both public internet and private/enterprise environments.**

3. **A capability token MUST NOT be treated as valid outside the distributed transaction that delegated it.**

> **NOTE:** OAuth, OIDC, SAML and bearer flows aren’t rejected — they simply become a subset of the broader model.

### Example: Delegated Capability

> You do not need to know the driver’s identity to start the engine.  
> You only need proof of delegated capability.

A **car key** represents a **capability token**:  
it enables the holder to perform a specific action without embedding personal identity.

In a distributed transaction model, a capability token is **valid only for the transaction in which it was delegated** (the authorized trip).  
Outside that transaction scope, the token is **inert**.

## Appendix

- **Transaction rails:** the transport and boundary surfaces that carry distributed execution (e.g., APIs, queues, brokers, message buses).
