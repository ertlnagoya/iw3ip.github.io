# SSI / DID / VC Basics

## Goal

- understand Self-Sovereign Identity (SSI)
- understand how DID and VC are used in policy decisions

## General Explanation

### Terms

- SSI: users manage identity credentials under user control
- DID: decentralized identifier (example: `did:example:alice`)
- VC: Verifiable Credential; here used as Consent VC

### Relationship Between the Three

- SSI is the design philosophy
- DID identifies a subject
- VC expresses verifiable claims about that subject

As a practical starting analogy, it is helpful to read this as: **DID = ID, VC = certificate, SSI = user-controlled identity architecture**.

### Why They Matter

In many conventional systems, one service provider stores both identity and permission data, making reuse and independent verification difficult.  
SSI-oriented systems aim to let users or organizations hold credentials and present them only when needed.

## Position in This System

### Minimal Model in This Site

- Consent VC includes `subject_did`, `dataset_id`, `allowed_purposes`, `valid_from/to`
- Data Publisher validates these fields and decides allow/deny

```mermaid
flowchart LR
A[Subject DID] --> B[Consent VC]
B --> C[Policy Engine]
C -->|allow| D[Send to Platform]
C -->|deny| E[Audit Log]
```

### Why It Works

- machine-checkable constraints: who, for what purpose, until when
- easier audit trace of policy decisions

### What This Sample Simplifies

- signature verification is still a placeholder
- DID resolution is not fully implemented
- advanced VC expression/presentation formats are not covered yet

### Future Extensions

- full signature verification (placeholder now)
- DID resolution against DID Documents
- PEP in front of publisher

## Sources

- W3C DID Core: <https://www.w3.org/TR/did-core/>
- W3C Verifiable Credentials Data Model 2.0: <https://www.w3.org/TR/vc-data-model-2.0/>
- DIF (Decentralized Identity Foundation): <https://identity.foundation/>
