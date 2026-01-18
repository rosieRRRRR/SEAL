# SEAL - Execution-Layer Mitigation for BIP-360

Encrypted Direct-to-Submission-Endpoint Execution for P2TSH

* Specification Version: 2.0.0
* Status: Draft - Implementation-ready specification, seeking public review
* Date: January 2026
* Author: rosiea
* Contact: [PQRosie@proton.me](mailto:PQRosie@proton.me)
* Licence: Apache License 2.0 - Copyright 2026 rosiea

---

## Abstract

SEAL is a voluntary execution-layer protocol for encrypted, direct-to-Submission-Endpoint delivery of fully signed Bitcoin transactions, where the Submission Endpoint is operated by a miner or mining pool. SEAL eliminates public mempool exposure prior to confirmation when private relay succeeds and defines execution-layer behavior only; its security posture is policy-defined.

BIP-360 (Pay-to-Tapscript-Hash) secures outputs at rest by removing Taproot key-path spending, but does not protect the execution window once a transaction is broadcast to the public mempool. SEAL addresses this execution-window exposure by cryptographically sealing and privately delivering signed transactions to a configured Submission Endpoint using ML-KEM-768, endpoint-signed key profiles, per-execution wallet pinning, and cryptographic SubmissionEvidence.

Under normal operation, SEAL does not broadcast transactions to the public mempool. If a transaction is decrypted and included but later orphaned or reorganized, the execution attempt is terminal and the spend is considered exposed. In such cases, the wallet transitions execution to a failure state, monitors the public mempool for conflicting transactions, and surfaces any observed double-spend attempts to the user or policy layer. Any recovery action, including fee-bumped replacement (RBF) or standard Bitcoin broadcast, is explicitly authorized and constitutes a new execution attempt.

SEAL introduces no Bitcoin consensus changes (beyond those defined by BIP-360 where applicable), no Script semantics changes, and no transaction validity changes (beyond those defined by BIP-360). It does not define endpoint discovery, inclusion guarantees, or privacy beyond execution-window exposure reduction.

---

## Executive Summary (Non-Normative)

### Encrypted Direct-to-Submission-Endpoint Execution for Post-Quantum Bitcoin Security

#### Problem

Bitcoin transactions are exposed during the execution window between signing and confirmation. Under standard operation, signed transactions are broadcast to the public mempool, making signatures observable prior to inclusion. A sufficiently capable quantum adversary could exploit this window to recover private keys and construct conflicting spends before confirmation.

BIP-360 hardens outputs at rest by removing Taproot key-path spending, but this protection does not extend into the execution window once a transaction is broadcast to the public mempool.

---

#### Approach

SEAL modifies transaction delivery during execution.

Instead of immediate public broadcast, wallets encrypt and submit fully signed transactions directly to a miner-operated Submission Endpoint. Transactions remain non-public during execution and become visible to the network only after confirmation. Submission Endpoints validate transaction integrity and return cryptographic SubmissionEvidence attesting to receipt and validation.

```
Standard execution:
Sign → Broadcast → Public Mempool → Confirm

SEAL execution:
Sign → Encrypt → Direct to Miner Submission Endpoint → Confirm
```

SEAL operates at the execution layer and composes with existing Bitcoin transaction formats, fee markets, and relay semantics.

---

#### Execution Discipline and Failure Semantics

SEAL enforces explicit execution discipline.

Each execution attempt is singular and terminal. Failure, exposure, or ambiguity halts execution and surfaces state.

If a transaction is decrypted and included but later orphaned or reorganized, the execution attempt is terminal and the spend is considered exposed. In such cases, wallets transition execution to a failure state, monitor the public mempool for conflicting transactions, and surface any observed double-spend attempts to the user or policy layer.

Recovery actions, including fee-bumped replacement (RBF) or standard Bitcoin broadcast, require explicit user or policy authorization and constitute new execution attempts.

---

#### Security Model

While private relay succeeds, SEAL prevents public mempool exposure of signed transaction bytes prior to confirmation, provides post-quantum confidentiality during submission, preserves transaction integrity via template hash binding, and supplies cryptographic accountability through SubmissionEvidence.

SEAL does not provide miner inclusion or timeliness guarantees, anonymity or metadata privacy, protection after abandonment of private execution, or on-chain post-quantum signatures. Transaction visibility during execution is reduced to the selected Submission Endpoint and does not eliminate miner visibility.

---

#### Deployment Posture

SEAL is opt-in and incrementally deployable. A single participating Submission Endpoint provides immediate benefit for high-risk transactions, while broader adoption improves availability and resilience without being required for correctness.

Public broadcast remains available through explicit authorization and is never triggered automatically.

---

#### Intended Use

SEAL is intended for execution contexts where exposure during the signing-to-confirmation window is unacceptable, including high-value settlement, institutional flows, and quantum-sensitive environments. It is not intended to replace standard Bitcoin broadcast for routine consumer payments.

---

## Index

0. [Scope](#0-scope)
1. [Terminology](#1-terminology)
   1.1 [Core Terms](#11-core-terms)
   1.2 [Participants](#12-participants)
   1.3 [Execution States](#13-execution-states)
   1.4 [Failure Modes (Normative)](#14-failure-modes-normative)
   1.5 [Cryptographic Terms](#15-cryptographic-terms)
2. [Design Goal](#2-design-goal)
3. [Execution Model Clarification (Normative)](#3-execution-model-clarification-normative)
4. [Why This Matters](#4-why-this-matters)
   4.1 [Execution-Window Exposure (Without SEAL)](#41-execution-window-exposure-without-seal)
   4.2 [Eliminating the Execution Window (With SEAL)](#42-eliminating-the-execution-window-with-seal)
   4.3 [Security Boundary Clarification](#43-security-boundary-clarification)
   4.4 [Liveness vs. Quantum Safety Tradeoff](#44-liveness-vs-quantum-safety-tradeoff)
5. [Non-Goals](#5-non-goals)
6. [System Assumptions](#6-system-assumptions)
   6.1 [Wallet Assumptions](#61-wallet-assumptions)
   6.2 [Submission Endpoint Assumptions](#62-submission-endpoint-assumptions)
   6.3 [Miner Identity and Trust Bootstrap](#63-miner-identity-and-trust-bootstrap)
   6.4 [Time Assumptions](#64-time-assumptions)
   6.5 [Failure Model Assumptions](#65-failure-model-assumptions)
   6.6 [Jurisdictional Assumptions](#66-jurisdictional-assumptions)
   6.7 [Funds at Rest Assumption](#67-funds-at-rest-assumption)
7. [Threat Model](#7-threat-model)
   7.1 [In-Scope Threats](#71-in-scope-threats)
   7.2 [Out-of-Scope Threats](#72-out-of-scope-threats)
   7.3 [Submission Endpoint Visibility Boundaries](#73-submission-endpoint-visibility-boundaries)
   7.4 [Economic Incentives, Abuse, and Correlation Risk](#74-economic-incentives-abuse-and-correlation-risk)
   7.4.1 [Submission Endpoint Incentives](#741-submission-endpoint-incentives)
   7.4.2 [Adversarial Submission Endpoint Behavior](#742-adversarial-submission-endpoint-behavior)
   7.4.3 [Correlation Risk and Multi-Endpoint Tradeoff](#743-correlation-risk-and-multi-endpoint-tradeoff)
   7.5 [Visibility Guarantees](#75-visibility-guarantees)
   7.6 [Residual Quantum Risk](#76-residual-quantum-risk)
   7.7 [Design Rationale and Common Objections](#77-design-rationale-and-common-objections)
8. [Protocol Overview](#8-protocol-overview)
9. [Protocol Operation](#9-protocol-operation)
   9.1 [Submission Endpoint Selection and Pinning](#91-submission-endpoint-selection-and-pinning)
   9.2 [MinerKEMProfile](#92-minerkemprofile)
   9.3 [Key Lifecycle and Revocation](#93-key-lifecycle-and-revocation)
   9.4 [Transaction Construction](#94-transaction-construction)
   9.5 [Template Hashing](#95-template-hashing)
   9.6 [Payload Encryption](#96-payload-encryption)
   9.6.1 [Expiry Enforcement](#961-expiry-enforcement)
   9.6.2 [Associated Authenticated Data](#962-associated-authenticated-data)
   9.6.3 [Encryption and Decryption Procedure](#963-encryption-and-decryption-procedure)
   9.6.4 [Encrypted Submission Object](#964-encrypted-submission-object)
   9.7 [Submission and SubmissionEvidence](#97-submission-and-submissionevidence)
   9.7.1 [Evidence Issuance Preconditions](#971-evidence-issuance-preconditions)
   9.7.2 [SubmissionEvidence Structure](#972-submissionevidence-structure)
   9.7.3 [Wallet Verification](#973-wallet-verification)
   9.8 [Execution State Observation](#98-execution-state-observation)
   9.9 [Failure Handling and Recovery](#99-failure-handling-and-recovery)
   9.10 [Multi-Endpoint Submission](#910-multi-endpoint-submission)
   9.11 [Replay Dampening](#911-replay-dampening)
   9.12 [Replay Cache Retention](#912-replay-cache-retention)
   9.13 [Local UTXO Selection Logic](#913-local-utxo-selection-logic)
   9.14 [Endpoint Offer Model](#914-endpoint-offer-model)
   9.15 [Wallet Economic Discipline](#915-wallet-economic-discipline)
   9.16 [Replace-By-Fee Under Private Relay](#916-replace-by-fee-under-private-relay)
   9.17 [Transport Profiles](#917-transport-profiles)
   9.18 [key_id Format Guidance](#918-key_id-format-guidance)
10. [Submission Endpoint Requirements](#10-submission-endpoint-requirements)
    10.1 [Identity Keys and Proof of Ownership (Normative)](#101-identity-keys-and-proof-of-ownership-normative)
    10.2 [MinerKEMProfile Publication (Normative)](#102-minerkemprofile-publication-normative)
    10.3 [Decryption and Validation (Normative)](#103-decryption-and-validation-normative)
    10.4 [Evidence Generation (Normative)](#104-evidence-generation-normative)
    10.5 [Mempool and Inclusion Behavior (Informative)](#105-mempool-and-inclusion-behavior-informative)
    10.6 [Failure and Rejection Scenarios (Normative)](#106-failure-and-rejection-scenarios-normative)
    10.6.1 [Rejection Classes (Normative)](#1061-rejection-classes-normative)
    10.6.2 [Rejection Response Format](#1062-rejection-response-format)
    10.7 [Submission Endpoint Clarification](#107-submission-endpoint-clarification)
11. [Security](#11-security)
    11.1 [Cryptographic Dependencies](#111-cryptographic-dependencies)
    11.2 [Execution Freezes](#112-execution-freezes)
    11.3 [Observer Failure Semantics](#113-observer-failure-semantics)
    11.4 [Epoch Clock Integration](#114-epoch-clock-integration)
    11.5 [Residual Quantum Exposure](#115-residual-quantum-exposure)
    11.6 [End-to-End Security Analysis](#116-end-to-end-security-analysis)
    11.7 [Security Summary](#117-security-summary)
12. [Deployment](#12-deployment)
    12.1 [Implementation Requirements Summary](#121-implementation-requirements-summary)
    12.1.1 [Wallet Integration Checklist (Informative)](#1211-wallet-integration-checklist-informative)
    12.2 [Submission Endpoint Deployment Checklist (Informative)](#122-submission-endpoint-deployment-checklist-informative)
    12.3 [Compatibility Notes](#123-compatibility-notes)
    12.4 [Deployment Strategy Guidance](#124-deployment-strategy-guidance)
13. [Test Vectors and Validation](#13-test-vectors-and-validation)
    13.1 [Template Hash Vectors (Deterministic)](#131-template-hash-vectors-deterministic)
    13.2 [Encryption Vectors](#132-encryption-vectors)
    13.3 [SubmissionEvidence Validation (Deterministic Structure)](#133-submissionevidence-validation-deterministic-structure)
    13.4 [Execution Scenarios (Informative)](#134-execution-scenarios-informative)
    13.5 [Validation Summary](#135-validation-summary)
14. [Formal Compliance](#14-formal-compliance)
15. [Explicit Non-Inclusions and Acknowledgments](#15-explicit-non-inclusions-and-acknowledgments)
16. [References](#16-references)

Annexes
Annex A. [SubmissionEvidence JSON Schema](#annex-a-submissionevidence-json-schema)
Annex B. [MinerKEMProfile JSON Schema](#annex-b-minerkemprofile-json-schema)
Annex C. [Execution State Transition Machine](#annex-c-execution-state-transition-machine)
Annex D. [KeyMail Integration](#annex-d-keymail-integration)
Annex E. [Relationship to PQSF](#annex-e-relationship-to-pqsf)
Annex F. [Reference Implementation Notes](#annex-f-reference-implementation-notes)
Annex G. [Future Work and Research Directions](#annex-g-future-work-and-research-directions)
Annex H. [Naming Consistency and Terminology Discipline](#annex-h-naming-consistency-and-terminology-discipline)
Annex I. [Relationship to Bitcoin Pre-Contracts (BPC)](#annex-i-relationship-to-bitcoin-pre-contracts-bpc)
Annex J. [Related and Linked Work](#annex-j-related-and-linked-work)


---

## 0. Scope

This specification defines SEAL: an execution-layer protocol for encrypted, direct-to-Submission-Endpoint delivery of fully signed Bitcoin transactions.

SEAL operates exclusively at the execution layer.

It does not:

* introduce consensus changes (beyond those defined by BIP-360 where applicable)
* modify Script semantics
* alter transaction validity rules (beyond those defined by BIP-360)
* introduce custody authority
* require network policy changes

SEAL is **policy-defined**. Its security properties depend on:

* wallet configuration
* Submission Endpoint selection
* user-authorized recovery actions

SEAL is a voluntary execution-layer protocol.

---

## 1. Terminology

### 1.1 Core Terms

SEAL
The execution-layer protocol defined in this specification.

Execution Window
The interval between transaction signing and on-chain confirmation during which signatures are observable.

Submission Endpoint
An identity-bound miner or pool service that publishes key profiles, accepts encrypted submissions, validates transactions, issues SubmissionEvidence, and may attempt inclusion in blocks. A Submission Endpoint does not imply control of physical hashpower; `miner_id` denotes endpoint identity only.

Private Relay
The intended execution path where transactions are submitted encrypted and remain non-public until confirmation.

Public Broadcast
Standard Bitcoin transaction propagation via the public mempool. Under SEAL, public broadcast is used only as an explicit recovery action after execution failure.

Abandonment of Private Execution
An exception-only recovery action where the user or policy explicitly authorizes transition from private relay to public broadcast, acknowledging quantum exposure.

Execution Freeze
A state where automatic execution progression halts pending explicit user or policy authorization.

Template Hash
A cryptographic commitment to the transaction structure used to verify integrity after decryption.

SubmissionEvidence
Cryptographic proof of submission and validation generated by the Submission Endpoint. Evidence proves submission, not inclusion.

---

### 1.2 Participants

Wallet
The software or system responsible for transaction construction, signing, encryption, submission, and execution state observation.

User
The entity controlling the wallet and authorizing execution decisions.

Submission Endpoint
The identity-bound service that receives encrypted submissions, validates them, and may include transactions in blocks.

Observer
The wallet-side component responsible for monitoring execution state, including mempool observation and blockchain reorganization detection.

---

### 1.3 Execution States

SEAL execution is governed by a strict, deterministic execution state machine. At any point in time, a transaction execution attempt MUST be in exactly one execution state.

The execution state machine defines allowed transitions and prohibits implicit retries, silent recovery, or automatic public broadcast. Loss of state, ambiguity, or observer failure is treated as execution failure.

Execution States:

* PENDING
  Transaction constructed and signed locally. No submission has occurred.

* SUBMITTED
  Encrypted submission accepted by a Submission Endpoint and valid `SubmissionEvidence` verified and persisted.

* CONFIRMED
  Transaction included in a block with sufficient confirmation depth.

* FAILED
  Execution failure detected. All automatic progression halts.

* AUTHORIZED_PUBLIC
  Private execution semantics explicitly abandoned by user or policy authorization. Public broadcast is permitted.

Allowed State Transitions:

| From      | To                | Condition                                            |
| --------- | ----------------- | ---------------------------------------------------- |
| PENDING   | SUBMITTED         | Verified `SubmissionEvidence` received and persisted |
| SUBMITTED | CONFIRMED         | Confirmation observed                                |
| SUBMITTED | FAILED            | Any failure condition or ambiguity                   |
| FAILED    | AUTHORIZED_PUBLIC | Explicit user or policy authorization                |

Prohibited Behavior:

Wallets and agents MUST NOT:

* automatically broadcast transactions publicly
* retry, resubmit, or fee-bump without explicit authorization
* re-enter `PENDING` or `SUBMITTED` for the same transaction instance
* advance execution state after restart without verified observation

The normative execution state transition machine is defined in Annex C.

---

### 1.4 Failure Modes (Normative)

Mempool Observation Failure
Inability to observe the public mempool or detect exposure.

Reorganization Failure
Transaction confirmed block becomes orphaned beyond recovery window.

Submission Rejection
Submission Endpoint refuses or fails to accept the encrypted transaction.

Timeout
Transaction not confirmed within policy-defined maximum wait time.

Exposure Detected
Transaction observed in public mempool despite private submission.

All failure modes result in transition to `FAILED` state and require explicit authorization for recovery.

---

### 1.5 Cryptographic Terms

ML-KEM-768
NIST-standardized post-quantum key encapsulation mechanism used to derive a shared secret between wallet and Submission Endpoint.

HKDF
HMAC-based Key Derivation Function used to derive an AEAD key from the ML-KEM shared secret.

AEAD  
Authenticated Encryption with Associated Data. SEAL uses XChaCha20-Poly1305 for payload encryption and metadata binding.

Hybrid Encryption
KEM-based key encapsulation followed by AEAD-based payload encryption.

miner_id
Submission Endpoint identifier. `miner_id` denotes endpoint identity only and MUST NOT be interpreted as a guarantee of physical hashpower, block production, or inclusion.

---

## 2. Design Goal

**Goal:** Eliminate public mempool exposure during the execution window for fully signed Bitcoin transactions by enabling encrypted, direct-to-Submission-Endpoint delivery with cryptographic accountability and deterministic failure handling.

Non-Goal: Providing miner inclusion guarantees, anonymity, or on-chain post-quantum signatures.

---

### 3. Execution Model Clarification (Normative)

SEAL operates after transaction construction and signing and before any public broadcast.

SEAL modifies only the execution phase of a Bitcoin transaction. It does not alter UTXO selection, transaction structure, signature generation, fee calculation, or consensus validation.

Execution under SEAL proceeds as follows:

1. The wallet constructs and signs a standard Bitcoin transaction.
2. The wallet encrypts the fully signed transaction.
3. The encrypted payload is submitted directly to a pinned Submission Endpoint.
4. SubmissionEvidence is verified and persisted.
5. The wallet observes confirmation or failure.

If execution fails, exposure is detected, or observation becomes unreliable, execution halts. Any retry, fee-bumped replacement, or public broadcast requires explicit authorization and constitutes a new execution attempt.

SEAL preserves unconditional user authority to broadcast transactions publicly but never exercises that authority automatically.


---

## 4. Why This Matters

### 4.1 Execution-Window Exposure (Without SEAL)

Under standard Bitcoin execution, a signed transaction is broadcast to the public mempool prior to confirmation. During this interval, transaction signatures are globally observable.

This execution window creates a vulnerability: a sufficiently capable quantum adversary observing the public mempool could recover private keys from exposed signatures and construct conflicting transactions before confirmation.

BIP-360 hardens outputs at rest by removing Taproot key-path spending. It does not protect signatures once a transaction is broadcast. Execution-window exposure remains.

---

### 4.2 Eliminating the Execution Window (With SEAL)

SEAL modifies transaction delivery during execution.

Instead of immediate public broadcast, the wallet encrypts the fully signed transaction and submits it directly to a selected Submission Endpoint. While private relay succeeds, signed transaction bytes are never broadcast to the public mempool and are not observable by public network participants.

Visibility during execution is constrained to the selected Submission Endpoint. Transactions become public only after confirmation or after explicit abandonment of private execution.

---

### 4.3 Security Boundary Clarification

SEAL addresses exposure during the execution window between transaction signing and on-chain confirmation.

SEAL does not provide protection after confirmation, does not prevent miner withholding or censorship, and does not eliminate miner visibility of transaction contents during private execution. Network-layer privacy, metadata protection, and correlation resistance are out of scope.

SEAL is an execution-layer mitigation. Output-layer hardening requires BIP-360 script-path-only outputs, and comprehensive post-quantum security ultimately requires consensus-layer upgrades.

---

### 4.4 Liveness vs. Quantum Safety Tradeoff

SEAL introduces an explicit tradeoff between execution liveness and execution-window exposure.

Standard Bitcoin broadcast maximizes liveness by allowing any miner to include a transaction, but exposes signatures publicly during execution. SEAL reduces execution-window exposure by constraining delivery to selected Submission Endpoints, which may delay or prevent inclusion if those endpoints refuse or withhold transactions.

This tradeoff is explicit and user-controlled. Public broadcast remains available only through explicit authorization and is never triggered automatically.

---

### 4.5 Default Behavior Guidance (Informative)

**Recommended wallet defaults:**

* Quantum-sensitive wallets: SEAL enabled by default
* General-purpose wallets: SEAL opt-in
* Always allow explicit abandonment of private execution

User control is paramount. Wallets MUST NOT lock users into private relay without an explicit escape path.

---

## 5. Non-Goals

SEAL does not provide miner inclusion guarantees, confirmation timeliness, anonymity, metadata privacy, or censorship resistance through private relay.

SEAL does not introduce on-chain post-quantum signatures, modify consensus rules (beyond those defined by BIP-360 where applicable), alter Script semantics, or change transaction validity (beyond those defined by BIP-360).

SEAL does not attempt to prevent miner withholding, coordination, or economic coercion. Such behavior is treated as an execution failure and surfaced for explicit recovery decisions.

---

## 6. System Assumptions

### 6.1 Wallet Assumptions

Wallets are assumed to:

* generate cryptographically secure random values
* implement Ed25519 and ML-KEM-768 correctly
* maintain secure local state
* perform mempool observation
* detect blockchain reorganizations
* enforce execution state machine transitions
* never silently retry or resubmit failed transactions

---

### 6.2 Submission Endpoint Assumptions

Submission Endpoints are assumed to:

* operate standard Bitcoin node software
* possess hashpower or pool membership (no minimum threshold)
* publish signed MinerKEMProfile with valid ML-KEM-768 public keys
* validate encrypted submissions
* generate cryptographic SubmissionEvidence
* not be trusted for inclusion guarantees

Submission Endpoints may:

* refuse transactions
* delay inclusion
* withhold transactions
* observe transaction contents

---

### 6.3 Miner Identity and Trust Bootstrap

Identity binding:
- Submission Endpoints are identified by `miner_id` (Ed25519 fingerprint)
- Identity is established via on-chain coinbase proof or external reputation

Trust model:
- Submission Endpoints are not trusted for privacy beyond execution window
- Submission Endpoints are not trusted for inclusion guarantees
- Users select Submission Endpoints based on reputation, hashrate, or policy

Out-of-band verification: Wallets SHOULD verify Submission Endpoint identity through:
- On-chain proof of mining activity
- Third-party attestation
- Community reputation signals

---

### 6.4 Time Assumptions

* Wallets have access to approximate wall-clock time
* Blockchain timestamps are used for confirmation depth
* Optional: Epoch Clock integration for high-assurance time binding

---

### 6.5 Failure Model Assumptions

* Execution failures are assumed to be permanent unless explicitly retried
* Ambiguous states default to `FAILED`
* Recovery always requires explicit authorization

---

### 6.6 Jurisdictional Assumptions

SEAL makes no assumptions about:

* legal frameworks
* regulatory compliance
* geographic restrictions

Implementers MUST evaluate local law independently.

---

### 6.7 Funds at Rest Assumption

For the primary post-quantum security posture, funds are assumed to be held in BIP-360 script-path-only outputs prior to execution.

SEAL does not require BIP-360 for correctness and MAY be used with any valid Bitcoin transaction. SEAL addresses exposure during execution only and does not provide post-confirmation protection.

For post-confirmation hardening, outputs SHOULD use BIP-360 script-path-only form and key reuse SHOULD be avoided.

---

## 7. Threat Model

### 7.1 In-Scope Threats

Quantum-capable mempool observer:
- Observes public mempool
- Extracts signatures
- Recovers private keys using quantum computer
- Constructs replacement transactions

Passive network observer:
- Intercepts encrypted submissions
- Attempts cryptanalysis (defended by ML-KEM-768)

Malicious Submission Endpoint:
- Withholds transactions
- Observes contents (unavoidable)
- Attempts correlation attacks

---

### 7.2 Out-of-Scope Threats

* Miner coordination or 51% attacks
* On-chain cryptanalysis post-confirmation
* Network-layer traffic analysis
* Endpoint discovery metadata
* Quantum attacks on ML-KEM-768 (assumed secure per NIST)

---

### 7.3 Submission Endpoint Visibility Boundaries

Submission Endpoints gain visibility into:

* Transaction structure
* Input and output addresses
* Signature material
* Fee amounts
* Timing metadata

Submission Endpoints cannot:

* Modify transactions (integrity protected by template_hash)
* Forge SubmissionEvidence (signature binding)
* Force inclusion (user retains broadcast authority)

---

### 7.4 Economic Incentives, Abuse, and Correlation Risk

SEAL introduces economic and correlation considerations by constraining transaction delivery to selected Submission Endpoints during execution. These effects are explicit and policy-controlled.

---

#### 7.4.1 Submission Endpoint Incentives

Submission Endpoints are economically incentivized through:

* standard Bitcoin transaction fees,
* optional service fees or premiums,
* reputational value from reliable execution and evidence issuance.

SEAL imposes no additional incentive mechanisms and does not alter the Bitcoin fee market. Endpoints are free to accept, delay, or refuse submissions based on their own policy.

---

#### 7.4.2 Adversarial Submission Endpoint Behavior

Submission Endpoints may act adversarially by:

* withholding or delaying transactions,
* selectively refusing submissions,
* observing transaction contents,
* collecting timing or metadata for correlation.

SEAL treats all such behavior as execution risk rather than protocol failure.

Mitigations include:

* explicit endpoint selection and pinning,
* optional multi-endpoint submission,
* SubmissionEvidence for auditability and accountability,
* preservation of explicit public broadcast authority.

No endpoint is trusted for inclusion, ordering, or censorship resistance.

---

#### 7.4.3 Correlation Risk and Multi-Endpoint Tradeoff

Using a single Submission Endpoint minimizes correlation surface but increases dependence on that endpoint’s availability and policy.

Using multiple endpoints may improve inclusion probability but increases correlation and metadata exposure. This tradeoff is explicit and user-controlled.

Multi-endpoint submission is optional, disabled by default, and requires explicit opt-in with user awareness of correlation risk.

Early deployment of SEAL submission endpoints is expected to occur through voluntary participation by individual miners or mining pools publishing Submission Endpoint interfaces and MinerKEMProfile objects via existing channels such as websites, documentation, or direct coordination with wallet implementers. No registry, discovery protocol, or coordinated rollout is required for initial adoption.

---

### **7.5 Visibility Guarantees**

While private relay succeeds, signed transaction bytes are not broadcast to the public mempool and are not observable by public network participants.

Transaction visibility during execution is limited to the selected Submission Endpoint. This represents a reduction in observer set size, not a guarantee of privacy, anonymity, or non-correlation.

If private execution is abandoned or exposure occurs, standard Bitcoin visibility assumptions apply immediately.

---

### 7.6 Residual Quantum Risk

Execution window: Eliminated (primary SEAL goal)

Post-confirmation: Outputs remain vulnerable unless BIP-360 script-path-only is used

Miner observation: Submission Endpoints observe transaction contents

Consensus layer: No on-chain post-quantum signatures (requires future upgrade)

---

### 7.7 Design Rationale and Common Objections

SEAL is an opt-in execution-layer protocol that reduces exposure during the signing-to-confirmation window without altering Bitcoin consensus (beyond those defined by BIP-360 where applicable), Script semantics, or transaction validity (beyond those defined by BIP-360).

Key design points:

* Execution-layer only: SEAL modifies delivery and execution discipline, not transaction structure or consensus rules.
* No inclusion guarantees: Submission Endpoints may withhold or refuse transactions. This is treated as execution failure and surfaced explicitly.
* Explicit recovery: All retries, fee bumps, or public broadcast require explicit user or policy authorization.
* Centralization tradeoff is explicit: Constraining delivery to selected endpoints reduces exposure but may reduce liveness. This tradeoff is user-controlled.
* Composability: SEAL composes with BIP-360 for output hardening and with other authorization frameworks without creating dependencies.

SEAL does not claim comprehensive post-quantum security. It addresses execution-window exposure only.

---

## 8. Protocol Overview

SEAL Execution Flow:

1. Endpoint Selection: Wallet selects and pins a Submission Endpoint
2. Key Retrieval: Wallet retrieves and verifies `MinerKEMProfile`
3. Transaction Construction: Standard Bitcoin transaction construction
4. Template Hashing: Wallet computes cryptographic commitment to transaction structure
5. Encryption: Wallet encrypts transaction using ML-KEM-768 hybrid encryption
6. Submission: Wallet submits encrypted payload to Submission Endpoint
7. Evidence: Submission Endpoint validates and returns signed `SubmissionEvidence`
8. **Observation:** Wallet monitors for confirmation or exposure
9. Confirmation: Transaction confirmed, execution complete
10. Failure Handling: On failure, wallet surfaces state and awaits explicit authorization

---

## 9. Protocol Operation

This section defines the execution behavior of SEAL from Submission Endpoint selection through execution completion or failure. All requirements expressed using RFC 2119 keywords are mandatory.

---

### 9.1 Submission Endpoint Selection and Pinning

Wallets MUST select a Submission Endpoint before transaction construction.

Selection criteria MAY include:

* hashrate or pool size
* reputation or reliability metrics
* advertised fee constraints
* geographic or jurisdictional preferences

Once selected, the endpoint identity (`miner_id`) MUST be pinned for the duration of the execution attempt.

Wallets MUST NOT silently switch Submission Endpoints mid-execution.

Automatic failover or substitution without explicit authorization is prohibited.

Multi-endpoint submission is permitted only as an explicit opt-in and is defined in Section 9.10.

---

### 9.2 MinerKEMProfile

Submission Endpoints publish encryption key material via a signed MinerKEMProfile.

A MinerKEMProfile contains:

```json
{
  "miner_id": "<hex fingerprint of Ed25519 identity key>",
  "key_id": "kem-2026-01",
  "kem_public_key": "<base64 ML-KEM-768 public key>",
  "valid_from": "<ISO 8601 timestamp>",
  "valid_until": "<ISO 8601 timestamp>",
  "signature": "<Ed25519 signature over canonical JSON>",
  "optional_metadata": {
    "endpoint_url": "https://...",
    "fee_offer": { },
    "recommended_expiry_seconds": 300,
    "max_clock_skew_seconds": 60
  }
}
```

Wallets MUST:

1. Verify the Ed25519 signature over canonical JSON.
2. Verify the current time is within the validity window.
3. Verify that `miner_id` matches the pinned endpoint identity.

Expired or revoked keys MUST NOT be used.

If present, `recommended_expiry_seconds` and `max_clock_skew_seconds` are advisory hints only. Wallets MAY use them to reduce edge-case expiry rejections, but MUST continue to enforce local execution policy and SHOULD keep expiry windows short.

Submission Endpoints SHOULD rotate ML-KEM-768 keys periodically.

---

### 9.3 Key Lifecycle and Revocation

KEM keys are valid only within their declared validity window.

Wallets MUST reject expired keys and SHOULD avoid submitting to keys nearing expiration.

Submission Endpoints MAY revoke keys by publishing an updated MinerKEMProfile with an earlier `valid_until` timestamp.

If a KEM private key is compromised, the endpoint MUST immediately revoke the corresponding profile and reject new submissions.

---

### 9.4 Transaction Construction

Transaction construction occurs entirely within the wallet prior to SEAL execution.

SEAL does not modify:

* UTXO selection
* input grouping
* output construction
* fee calculation
* signature generation

For post-confirmation quantum hardening, wallets SHOULD use BIP-360 script-path-only outputs.

---

### 9.5 Template Hashing

The template hash is a cryptographic commitment to the exact transaction bytes intended for execution.

```
template_hash = SHA256(raw_transaction_bytes)
```

The input MUST be the fully serialized Bitcoin transaction, including witness data.

The template hash is:

* included in encrypted submission metadata
* verified by the Submission Endpoint after decryption
* used to detect tampering, corruption, or substitution

If the recomputed template hash does not match, the submission MUST be rejected.

---

### 9.6 Payload Encryption

SEAL uses hybrid encryption to keep the full signed transaction payload confidential prior to confirmation.

Hybrid encryption consists of:

1. ML-KEM-768 to derive a shared secret
2. AEAD to encrypt the transaction bytes with authenticated binding to submission metadata

The plaintext encrypted payload is the raw, fully signed Bitcoin transaction bytes, including witness data.

### 9.6.1 Expiry Enforcement

Each encrypted submission includes an `expires_at` timestamp defining the latest time at which the submission remains valid.

A submission is considered received at the time the Submission Endpoint application accepts the submission for processing.

Submissions received after `expires_at` MUST be rejected and MUST NOT be decrypted, validated, or issued SubmissionEvidence. Expiry enforcement applies regardless of transport delays, queueing, or retry behavior. Submission Endpoints evaluate `expires_at` against their local wall-clock time.

Wallets SHOULD select expiry windows that are short relative to the execution policy to bound replay and delayed-delivery risk. Wallets SHOULD also account for clock skew and transport latency when setting `expires_at`, ensuring sufficient allowance to avoid edge-case rejection caused by time drift between wallet and Submission Endpoint.

#### 9.6.2 Associated Authenticated Data

AAD MUST include at minimum:

* miner_id
* key_id
* submission_id
* template_hash
* expires_at

AAD MUST be encoded in a canonical, byte-stable form.

#### 9.6.3 Encryption and Decryption Procedure

Wallet procedure:

1. ML-KEM encapsulation to derive a shared secret.
2. HKDF key derivation using context `"seal/aead"`.
3. Fresh 24-byte nonce generation.
4. XChaCha20-Poly1305 encryption of the transaction bytes with canonical AAD.

Endpoint procedure:

1. ML-KEM decapsulation.
2. Identical HKDF derivation.
3. Canonical AAD reconstruction.
4. AEAD decryption and authentication.
5. Template hash recomputation and verification.

Any failure MUST result in rejection.

#### 9.6.4 Encrypted Submission Object

The submission object MUST contain:

* miner_id
* key_id
* submission_id
* template_hash
* expires_at
* kem_ciphertext
* nonce
* ciphertext
* tag

No transformation of transaction bytes is permitted.

---

### 9.7 Submission and SubmissionEvidence

SubmissionEvidence attests only to successful decryption and validation of an encrypted submission by a Submission Endpoint.

SubmissionEvidence does not imply miner intent, ordering, priority, inclusion guarantees, censorship resistance, or confirmation likelihood.

#### 9.7.1 Evidence Issuance Preconditions

A Submission Endpoint MUST issue SubmissionEvidence only after:

1. Successful decryption and authentication
2. Template hash verification
3. Bitcoin consensus validation
4. Confirmation that the transaction is not already confirmed
5. Replay checks if implemented

### 9.7.2 SubmissionEvidence Structure

SubmissionEvidence MUST be a signed object containing at minimum:

```json
{
  "submission_id": "<uuid>",
  "miner_id": "<endpoint identifier>",
  "key_id": "<kem key identifier>",
  "template_hash": "<sha256>",
  "timestamp": "<ISO 8601>",
  "status": "accepted",
  "validation_proof": {
    "template_hash_verified": true,
    "consensus_valid": true,
    "not_in_blockchain": true,
    "replay_checked": true
  },
  "optional": {
    "endpoint_time": "<ISO 8601>"
  },
  "signature": "<ed25519 signature>"
}
```

All fields except `signature` MUST be covered by the signature. If present, `optional.endpoint_time` is advisory diagnostic information only and has no execution semantics.

#### 9.7.3 Wallet Verification

Wallets MUST verify:

1. Signature validity
2. Submission ID match
3. Template hash match
4. Key ID consistency
5. Validation flags

Failure of any check MUST transition execution to FAILED.

---

### 9.8 Execution State Observation

Wallets MUST observe execution state continuously after entering SUBMITTED until transition to CONFIRMED or FAILED.

Wallets MUST observe:

* blockchain confirmation state
* public mempool exposure
* observer health and connectivity

Unexpected public exposure MUST cause immediate transition to FAILED.

Loss of confirmation due to reorganization MUST transition execution to FAILED.

---

### 9.9 Failure Handling and Recovery

Any execution failure, exposure event, timeout, reorganization, or observation ambiguity is terminal for the execution attempt.

FAILED execution attempts MUST NOT resume or implicitly retry.

Any retry, fee-bumped replacement, or public broadcast requires explicit user or policy authorization and constitutes a new execution attempt.

---

### 9.10 Multi-Endpoint Submission

Multi-endpoint submission allows parallel submission to multiple endpoints.

This feature is optional and disabled by default.

* Explicit opt-in is required.
* Identical transaction bytes and template hash MUST be used.
* Exposure at any endpoint is a global failure.
* Confirmation by any endpoint finalizes execution.

---

### 9.11 Replay Dampening

Endpoints MAY implement replay dampening keyed by submission_id.

Replay dampening is advisory only.

---

### 9.12 Replay Cache Retention

Replay cache entries SHOULD be retained for 24 to 48 hours.

---

### 9.13 Local UTXO Selection Logic

UTXO selection occurs entirely within the wallet.

SEAL does not constrain coin selection strategy.

---

### 9.14 Endpoint Offer Model

Wallets select a feerate locally, then choose a compatible endpoint.

SEAL defines no speed classes or priority tiers.

---

### 9.15 Wallet Economic Discipline

Wallets SHOULD:

* display total fees clearly
* surface premiums or surcharges
* cap fees relative to public estimates

---

### 9.16 Replace-By-Fee Under Private Relay

Each fee-bumped transaction constitutes a new execution attempt.

Each RBF attempt MUST use a new `submission_id` and MUST result in distinct SubmissionEvidence if accepted.

SubmissionEvidence MUST NOT be reused across attempts.

Note: This section describes fee-bumped replacement while executing under Private Relay prior to confirmation. Under SEAL, any fee-bumped replacement is a new execution attempt and MUST use a new `submission_id` with distinct SubmissionEvidence if accepted. This is distinct from the post-orphaning or reorganization scenario described in the Abstract and Executive Summary: if a transaction is confirmed and later orphaned or reorganized, the prior execution attempt is terminal and any recovery action, including RBF or public broadcast, requires explicit user or policy authorization and constitutes a new execution attempt.

---

### 9.17 Transport Profiles

SEAL assumes authenticated transport.

HTTPS is common. TLS provides defense in depth only.

KeyMail MAY be used as an optional asynchronous transport.

---

### 9.18 key_id Format Guidance

The key_id identifies the specific ML-KEM public key used for encryption.

The value is opaque and MUST NOT be interpreted as conveying security properties.

Structured formats reflecting key rotation epochs are recommended.

---

## 10. Submission Endpoint Requirements

This section defines the required and permitted behavior of Submission Endpoints.

Submission Endpoints are boundary-limited trusted components: they are relied
upon only within explicitly defined execution-layer boundaries.

Submission Endpoints are NOT trusted for:

- inclusion guarantees,
- censorship resistance,
- permanent privacy,
- ordering or priority,
- timeliness guarantees.

---

### 10.1 Identity Keys and Proof of Ownership (Normative)

Submission Endpoints MUST maintain a long-term Ed25519 identity key pair.

The identity key is used solely for:

- signing MinerKEMProfile objects,
- signing SubmissionEvidence objects,
- endpoint attribution and auditability.

The identity key MUST NOT be used for:

- Bitcoin transaction signing,
- consensus participation,
- block template construction.

Identity binding:

Submission Endpoints MUST provide a publicly verifiable proof that the identity
key corresponds to the operator of the endpoint.

Acceptable proofs MAY include:

- coinbase transaction attribution,
- publicly documented mining pool identity,
- long-standing operational reputation,
- other widely verifiable association.

The exact proof mechanism is out of scope for SEAL, but MUST be sufficient for
wallets to pin and audit endpoint identity.

---

### 10.2 MinerKEMProfile Publication (Normative)

Submission Endpoints MUST generate ML-KEM-768 key pairs for payload encryption.

Submission Endpoints MUST publish a signed MinerKEMProfile containing at minimum:

```json
{
  "miner_id": "<endpoint identifier>",
  "key_id": "kem-2026-01",
  "kem_public_key": "<base64>",
  "valid_from": "<ISO 8601 timestamp>",
  "valid_until": "<ISO 8601 timestamp>",
  "signature": "<ed25519 signature>"
}
```

Rules:

* MinerKEMProfile MUST be signed with the endpoint’s Ed25519 identity key.
* Wallets MUST verify the signature and validity window before use.
* Expired or revoked keys MUST NOT be used for new submissions.
* Submission Endpoints SHOULD rotate KEM keys periodically.

---

### 10.3 Decryption and Validation (Normative)

Upon receiving an encrypted submission, the Submission Endpoint MUST perform the
following steps in order:

1. Decapsulate the ML-KEM ciphertext to derive the shared secret.
2. Derive the AEAD key using HKDF with the specified context.
3. Reconstruct canonical AAD exactly as specified by the wallet.
4. Perform AEAD authentication and decryption.
5. Verify that the decrypted transaction bytes match `template_hash`.
6. Validate the transaction against Bitcoin consensus rules.
7. Verify that the transaction is not already confirmed on-chain.
8. Apply replay cache checks if implemented.

If any step fails, the Submission Endpoint MUST reject the submission.

Successful completion of all steps is a prerequisite for issuing
SubmissionEvidence.

---

### 10.4 Evidence Generation (Normative)

Submission Endpoints MUST issue SubmissionEvidence only after successful
completion of all required validation steps.

SubmissionEvidence MUST:

* be signed with the endpoint’s identity key,
* accurately reflect the validation actions performed,
* include a bounded validation proof,
* include a timestamp indicating evidence issuance time.

SubmissionEvidence MUST NOT:

* imply inclusion intent,
* imply ordering or priority,
* imply censorship resistance,
* imply permanent privacy guarantees.

---

### 10.5 Mempool and Inclusion Behavior (Informative)

Submission Endpoints MAY:

* hold accepted transactions in a private queue,
* attempt inclusion in a candidate block,
* delay inclusion,
* refuse inclusion entirely.

Submission Endpoints are under no obligation to include any transaction.

SEAL does not impose inclusion requirements, penalties, or enforcement
mechanisms on endpoints.

---

### 10.6 Failure and Rejection Scenarios (Normative)

Submission Endpoints MUST reject encrypted submissions that fail validation or
policy checks.

Rejections MUST be explicit, bounded, and non-oracular.

---

#### 10.6.1 Rejection Classes (Normative)

Submission Endpoints MUST classify rejections using one of the following
`rejection_code` values:

* `DECRYPTION_FAILED`
* `TEMPLATE_HASH_MISMATCH`
* `INVALID_TRANSACTION`
* `REPLAY_DETECTED`
* `KEY_EXPIRED_OR_REVOKED`
* `POLICY_REJECTED`
* `ALREADY_CONFIRMED`
* `ALREADY_IN_MEMPOOL`
* `INTERNAL_ERROR`

No other rejection codes are permitted.

---

### 10.6.2 Rejection Response Format

Rejection responses MUST conform to the following structure:

```json
{
  "status": "rejected",
  "rejection_code": "TEMPLATE_HASH_MISMATCH",
  "timestamp": "<ISO 8601 timestamp>",
  "optional": {
    "endpoint_time": "<ISO 8601>"
  },
  "details": "optional, human-readable"
}
```

Rules:

* `rejection_code` MUST be one of the bounded values in Section 10.6.1.
* `details` is OPTIONAL and MUST NOT disclose cryptographic material, internal policy thresholds, or system internals.
* If present, `optional.endpoint_time` is advisory diagnostic information only and MUST NOT be interpreted as an inclusion signal or guarantee.
* Rejection responses SHOULD be authenticated where the transport supports it.

A rejection terminates the execution attempt unless explicit user or policy authorization permits retry.

---

### 10.7 Submission Endpoint Clarification

A Submission Endpoint is an execution-layer interface operated by a miner or pool that publishes authenticated encryption key material, accepts encrypted transaction submissions, validates transaction integrity and consensus correctness, and may attempt transaction inclusion.

The identifier `miner_id` denotes endpoint identity only. It MUST NOT be interpreted as a guarantee of physical hashpower, block production, inclusion likelihood, ordering, or censorship resistance.

Submission Endpoints may delay, withhold, or refuse transactions without violating SEAL.

---

## 11. Security

This section defines the security properties provided by SEAL, the explicit
boundaries of those properties, and the residual risks that remain.

SEAL is an **execution-layer mitigation**. It reduces exposure during the
execution window only and does not claim comprehensive post-quantum security.

---

### 11.1 Cryptographic Dependencies

SEAL relies on the following cryptographic primitives:

Required algorithms:

- ML-KEM-768  
  Post-quantum key encapsulation mechanism (NIST FIPS 203), used to derive a
  shared secret between wallet and Submission Endpoint.

- HKDF (SHA-256)  
  Key derivation function used to derive an AEAD key from the ML-KEM shared
  secret.

- XChaCha20-Poly1305  
  Authenticated encryption with associated data (AEAD) used to encrypt the full
  signed transaction payload and bind submission metadata.

- Ed25519  
  Digital signature algorithm used for endpoint identity, MinerKEMProfile
  signatures, and SubmissionEvidence signatures.

- SHA-256  
  Hash function used for `template_hash` computation.

Security notes:

- Ed25519 is a classical signature scheme and is not relied upon for
  post-quantum confidentiality.
- Compromise of Ed25519 keys affects attribution and auditability only, not
  transaction confidentiality during the execution window.

---

### **11.2 Execution Freezes**

An execution freeze occurs when execution cannot safely proceed without risking unexpected exposure or ambiguity.

Execution freezes on detection of public exposure, observer failure, submission rejection, timeout, or reorganization. During a freeze, automatic progression halts and execution state is surfaced for explicit user or policy decision.

Execution freezes do not imply inclusion failure or recovery intent. Any subsequent action requires explicit authorization and constitutes a new execution attempt.

---

### **11.3 Observer Failure Semantics**

Reliable observation of execution state is security-critical.

Observer failure includes loss of blockchain connectivity, inability to query confirmation state, mempool observation unavailability, or divergence between configured observation sources.

On observer failure, the wallet MUST attempt recovery by re-establishing connectivity and querying transaction status. If execution status cannot be determined safely, execution transitions to FAILED and state is surfaced for explicit user or policy decision.

Observer failure MUST NOT trigger automatic retry, fee bumping, or public broadcast.

---

### 11.4 Epoch Clock Integration

Wallets and Submission Endpoints operating in high-accountability environments may bind SubmissionEvidence to Epoch Clock ticks.

Epoch Clock binding provides tamper-evident evidence timestamps and reduces temporal ambiguity during audit or dispute analysis. Binding is optional and additive.

Epoch Clock integration does not define or replace expiry enforcement. Unless explicitly specified by an implementation profile, Submission Endpoints enforce `expires_at` using local wall-clock time. Implementations requiring security-critical timing MAY express validity windows in terms of Epoch Clock ticks and treat wall-clock time as advisory only; such enforcement semantics are outside the scope of SEAL conformance.

---

### **11.5 Residual Quantum Exposure**

SEAL eliminates public mempool exposure during the execution window only.

Residual risks include visibility of transaction contents to Submission Endpoints during private execution, post-confirmation exposure if non-BIP-360 outputs are reused, and the absence of consensus-layer post-quantum signatures.

Mitigation of these risks requires output-layer hardening with BIP-360, avoidance of key and address reuse, and future consensus-layer upgrades.

---

### **11.6 End-to-End Security Analysis**

SEAL reduces execution-window exposure by preventing public mempool visibility of signed transactions prior to confirmation when private relay succeeds.

A quantum adversary observing the public mempool is denied access to signatures during execution. An adversary intercepting encrypted submissions is constrained by the security properties of ML-KEM-768 and AEAD. Submission Endpoints may observe transaction contents but cannot modify transactions, forge SubmissionEvidence, or compel execution outcomes.

If execution fails, exposure occurs, or observation becomes unreliable, execution halts and state is surfaced for explicit recovery decisions. No automatic retry, fee bumping, or public broadcast occurs.

SEAL preserves Bitcoin’s consensus guarantees by ensuring that all transactions remain valid under standard rules and that public broadcast remains available through explicit authorization.

---

### 11.7 Security Summary

SEAL reduces exposure during the execution window by constraining transaction delivery and enforcing explicit execution discipline.

While private relay succeeds, signed transaction bytes are not broadcast to the public mempool and are not observable by public network participants. Visibility during execution is limited to the selected Submission Endpoint.

SEAL does not guarantee miner inclusion, confirmation timeliness, ordering, censorship resistance, anonymity, metadata privacy, or post-confirmation protection. Submission Endpoints may withhold, delay, or refuse transactions, and may observe transaction contents during private execution.

SEAL does not provide on-chain post-quantum signatures and does not alter Bitcoin consensus rules (beyond those defined by BIP-360 where applicable), Script semantics, or transaction validity (beyond those defined by BIP-360). Post-confirmation hardening requires output-layer measures such as BIP-360, and comprehensive post-quantum security ultimately requires consensus-layer upgrades.

On any failure, exposure, reorganization, or observation ambiguity, execution halts and state is surfaced for explicit user or policy decision. No automatic retry, fee bumping, or public broadcast occurs.

SEAL preserves user sovereignty by ensuring that all recovery actions are explicit, authorized, and treated as new execution attempts, while remaining fully compatible with existing Bitcoin infrastructure.

---

## 12. Deployment

SEAL is designed for incremental, opt-in deployment without requiring Bitcoin consensus changes, miner coordination, or network policy modification. Deployment choices are policy-defined and user-controlled.

---

### 12.1 Implementation Requirements Summary

A SEAL-compliant deployment MUST satisfy the following requirements.

**Wallet requirements:**

* Implement the execution state machine exactly as specified.
* Persist execution state and SubmissionEvidence durably across restarts.
* Enforce execution freezes on failure, exposure, or ambiguity.
* Implement continuous observation of confirmation state, mempool exposure, and reorganizations.
* Verify MinerKEMProfile signatures and enforce key validity windows.
* Compute and verify template hashes exactly.
* Verify SubmissionEvidence signatures and contents before any execution state transition.
* Never automatically retry, fee-bump, resubmit, or broadcast publicly.
* Require explicit user or policy authorization for all recovery actions.

**Submission Endpoint requirements:**

* Maintain a long-term Ed25519 identity key.
* Publish ML-KEM-768 public keys via signed MinerKEMProfile objects.
* Enforce key validity windows and reject expired or revoked keys.
* Perform full decryption and validation of encrypted submissions.
* Reconstruct canonical associated data exactly.
* Verify template hashes and Bitcoin consensus validity.
* Issue SubmissionEvidence only after all validation steps succeed.
* Reject invalid, replayed, or policy-disallowed submissions deterministically.
* Avoid issuing ambiguous or misleading evidence.

---

### 12.1.1 Wallet Integration Checklist (Informative)

- [ ] Implement the execution state machine exactly as specified.
- [ ] Persist execution state and SubmissionEvidence durably across restarts.
- [ ] Enforce execution freezes on failure, exposure, or ambiguity.
- [ ] Implement continuous observation of confirmation state, mempool exposure, and reorganizations.
- [ ] Verify MinerKEMProfile signatures and enforce key validity windows.
- [ ] Compute and verify template hashes over fully serialized transaction bytes.
- [ ] Verify SubmissionEvidence signatures and contents before any execution state transition.
- [ ] Never automatically retry, fee-bump, resubmit, or broadcast transactions publicly.
- [ ] Require explicit user or policy authorization for all recovery actions.

---

### 12.2 Submission Endpoint Deployment Checklist (Informative)

#### Cryptographic Discipline

- [ ] Maintain a long-term Ed25519 identity key.
- [ ] Publish signed MinerKEMProfile objects with ML-KEM-768 public keys.
- [ ] Enforce key validity windows and reject expired or revoked keys.
- [ ] Perform full ML-KEM decapsulation and AEAD authentication.
- [ ] Reconstruct canonical AAD exactly as provided by the wallet.
- [ ] Verify `template_hash` against decrypted transaction bytes.
- [ ] Validate transactions against Bitcoin consensus rules.
- [ ] Check for prior confirmation or replay before acceptance.
- [ ] Issue SubmissionEvidence only after all validation checks succeed.

#### Operational Discipline

- [ ] Maintain a bounded replay cache if replay dampening is implemented.
- [ ] Reject invalid or replayed submissions deterministically.
- [ ] Avoid issuing evidence for transactions that cannot be fully validated.
- [ ] Avoid issuing ambiguous, partial, or misleading SubmissionEvidence.

#### Accountability (Recommended)

- [ ] Bind SubmissionEvidence to Epoch Clock where available.
- [ ] Retain SubmissionEvidence for audit and dispute analysis.
- [ ] Track inclusion outcomes to detect systematic execution failures.

---

### 12.3 Compatibility Notes

SEAL operates entirely at the execution layer and requires no changes to Bitcoin consensus rules (beyond those defined by BIP-360 where applicable), Script semantics, mempool policy, or transaction validity (beyond those defined by BIP-360).

Transactions executed under SEAL are standard Bitcoin transactions. If private execution is explicitly abandoned, transactions may be broadcast publicly and processed under normal Bitcoin relay and confirmation semantics.

SEAL composes with BIP-360 but does not depend on it for correctness. BIP-360 script-path-only outputs are recommended for post-confirmation hardening, while SEAL addresses exposure during execution.

---

### 12.4 Deployment Strategy Guidance

SEAL may be deployed selectively for high-value or quantum-sensitive transactions using a limited set of Submission Endpoints.

Broader deployment may incorporate multiple endpoints to improve availability, with an associated increase in correlation surface. Endpoint selection, timeout configuration, retry strategy, and recovery behavior are policy-defined and user-controlled.

Public broadcast remains available only through explicit authorization and is never triggered automatically.

---

## 13. Test Vectors and Validation

This section defines test vectors and validation guidance to assist implementers
in verifying correctness of SEAL components.

Test material in this section is divided into:

- Deterministic vectors for hashing and canonical encoding
- Structural vectors for encryption and submission flows
- Scenario vectors for execution-state behavior

Unless explicitly stated, test vectors are illustrative and are not intended
to mandate byte-for-byte ciphertext equality across implementations.

---

### 13.1 Template Hash Vectors (Deterministic)

The template hash is a deterministic SHA-256 hash of the raw, fully serialized
Bitcoin transaction bytes, including witness data.

Input transaction (hex):

```

020000000001...

```

This byte sequence MUST be the exact serialized transaction as constructed
by the wallet.

Computation:

```

template_hash = SHA256(raw_transaction_bytes)

```

Expected output (hex):

```

e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855

```

Any deviation in transaction serialization or hashing MUST be treated as a
failure.

---

### 13.2 Encryption Vectors

This section provides illustrative encryption vectors for SEAL’s hybrid encryption process. These vectors validate structural correctness, binding behavior, and cryptographic flow, not byte-exact reproducibility.

Implementations MAY use these vectors for testing. Exact ciphertext values will differ unless all randomness sources are fixed.

---

ML-KEM-768 public key (base64):

```
<base64-encoded ML-KEM-768 public key>
```

This key is published by the Submission Endpoint in its `MinerKEMProfile` and is authenticated via an Ed25519 identity signature.

---

Plaintext transaction:

The plaintext encrypted payload is the raw, fully signed Bitcoin transaction bytes, including witness data.

```
020000000001...
```

No transformation, truncation, or hashing of the transaction bytes is performed prior to encryption.

---

Associated Authenticated Data (AAD):

The following fields MUST be bound as AEAD associated data using canonical encoding:

```json
{
  "miner_id": "a4f9c2e8…",
  "key_id": "kem-2026-01",
  "submission_id": "6b9f1d2c-3e4a-4c88-9a9e-21b6a0e5f3a2",
  "template_hash": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
  "expires_at": "2026-01-31T00:00:00Z"
}
```

AAD MUST be encoded in a byte-stable canonical form.
RFC 8785 (JSON Canonicalization Scheme) is RECOMMENDED.

Any modification to AAD fields or canonicalization MUST cause AEAD authentication failure.

---

Expected encapsulated key:

```
<base64-encoded ML-KEM ciphertext>
```

The encapsulated key is produced by ML-KEM-768 encapsulation using the Submission Endpoint’s public key. The derived shared secret MUST NOT be transmitted or logged.

---

Expected AEAD ciphertext:

```
<base64-encoded XChaCha20-Poly1305 ciphertext>
```

Encryption is performed using:

* ML-KEM-768 for key encapsulation
* HKDF for symmetric key derivation
* XChaCha20-Poly1305 for AEAD payload encryption

The AEAD nonce MUST be 24 bytes and randomly generated per submission attempt.

---

Decryption and verification requirements:

Upon receipt, the Submission Endpoint MUST:

1. Decapsulate the ML-KEM ciphertext to recover the shared secret
2. Derive the AEAD key via HKDF
3. Reconstruct canonical AAD exactly
4. Perform AEAD decryption and authentication
5. Recompute `template_hash` over decrypted transaction bytes
6. Reject the submission if any step fails

Any failure MUST result in rejection without partial acceptance.

---

Security properties validated by these vectors:

These vectors demonstrate that:

* The entire signed transaction payload is confidential prior to confirmation
* Hashing alone is insufficient to prevent exposure
* Any mutation of transaction bytes, metadata, or binding fields causes deterministic failure
* Transaction hex is never exposed to the public mempool during successful private relay

---

Determinism note:

These vectors are structural, not deterministic.

To produce reproducible ciphertext for testing, implementations MAY fix:

* ML-KEM randomness
* HKDF parameters
* AEAD nonce

Production implementations MUST use fresh randomness for every submission attempt.

---

### 13.3 SubmissionEvidence Validation (Deterministic Structure)

SubmissionEvidence objects MUST be signed over canonical JSON encoding excluding
the `signature` field.

Example evidence input fields:

```json
{
  "submission_id": "6b9f1d2c-3e4a-4c88-9a9e-21b6a0e5f3a2",
  "miner_id": "a4f9c2e8…",
  "key_id": "kem-2026-01",
  "template_hash": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
  "evidence_timestamp": "2026-01-17T12:00:00Z",
  "status": "ACCEPTED",
  "validation_class": "DECRYPTED_HASH_AND_TX_VALIDATED"
}
```

Validation requirements:

* Ed25519 signature MUST verify under the pinned endpoint identity key
* Canonical encoding MUST be enforced
* Any mutation of fields MUST invalidate the signature

SubmissionEvidence proves receipt and the asserted validation class only. It does
not imply inclusion intent or guarantee.

---

### 13.4 Execution Scenarios (Informative)

Implementations SHOULD validate behavior against the following execution
scenarios.

#### Scenario 1: Successful Private Relay

1. Wallet constructs and signs transaction
2. Wallet encrypts payload and submits to endpoint
3. Endpoint validates and returns SubmissionEvidence
4. Wallet observes confirmation

Expected execution state transitions:

```
PENDING → SUBMITTED → CONFIRMED
```

No public mempool exposure occurs.

---

#### Scenario 2: Submission Endpoint Rejection or Timeout

1. Wallet submits encrypted payload
2. Endpoint rejects or fails to respond
3. Wallet transitions to FAILED
4. Wallet awaits explicit recovery authorization

Expected execution state:

```
PENDING → SUBMITTED → FAILED
```

No automatic retry or broadcast occurs.

---

#### Scenario 3: Integrity Failure

1. Wallet submits encrypted payload
2. Endpoint decrypts but `template_hash` does not match
3. Endpoint rejects submission
4. Wallet enters FAILED state

Expected outcome:

* No SubmissionEvidence issued
* No partial acceptance
* No retry without explicit authorization

---

#### Scenario 4: Observer Failure with Confirmed Transaction

1. Wallet submits transaction
2. Endpoint includes transaction in a block
3. Wallet observer temporarily fails
4. Wallet recovers and detects confirmation

Expected behavior:

* Wallet transitions to CONFIRMED
* Wallet MUST NOT enter FAILED if confirmation is detected

---

#### Scenario 5: Unexpected Public Exposure

1. Wallet submits via private relay
2. Transaction appears in public mempool
3. Wallet detects exposure

Expected execution state:

```
SUBMITTED → FAILED
```

Wallet MUST surface exposure and require explicit recovery authorization.

---

### 13.5 Validation Summary

An implementation is considered correct if:

* Deterministic components (hashing, canonicalization, signatures) match exactly
* Encryption and decryption succeed with correct binding
* Any deviation causes deterministic failure
* Execution state transitions follow the normative state machine
* No automatic public broadcast occurs under any failure condition

---

### 14. Formal Compliance

A SEAL-compliant implementation MUST:

1. Implement the execution state machine as specified and persist execution state durably.
2. Use ML-KEM-768, HKDF, and XChaCha20-Poly1305 exactly as defined for payload encryption.
3. Validate MinerKEMProfile signatures and enforce key validity windows.
4. Compute and verify template hashes over fully serialized transaction bytes.
5. Verify SubmissionEvidence signatures and contents before any execution state transition.
6. Observe confirmation state, mempool exposure, and reorganizations continuously.
7. Halt execution on failure, exposure, or ambiguity and surface state explicitly.
8. Require explicit user or policy authorization for any retry, fee-bumped replacement, or public broadcast.
9. Treat all recovery actions as new execution attempts.
10. Never automatically retry, resubmit, or broadcast transactions publicly.

A SEAL-compliant implementation MAY:

* Support optional multi-endpoint submission if explicitly enabled.
* Integrate optional Epoch Clock binding.
* Use alternative authenticated transports such as KeyMail.
* Implement additional policy-layer controls outside the scope of this specification.

No behavior not explicitly permitted by this section is implied.

---

## 15. Explicit Non-Inclusions and Acknowledgments

SEAL explicitly does not define:

* Endpoint discovery mechanisms
* Miner registries or marketplaces
* Mandatory KeyMail usage
* Custody authority or quorum logic
* UI tier requirements
* Enforcement outside explicit user or policy authorization

This specification builds on prior research and discussion around Bitcoin execution security, post-quantum cryptography, and mempool exposure risks.

Thanks to:

* the BIP-360 authors and reviewers for advancing output-layer quantum hardening; special thanks to Hunter Beast for identifying key gaps in v1. Several cryptographic protections in this version were developed in response to those observations.
* Bitcoin Core developers and researchers for sustained work on mempool behavior and execution semantics,
* post-quantum cryptography contributors whose work informs practical transport-layer defenses,
* reviewers and operators who provided critical feedback on miner interaction, public broadcast authorization behavior, and threat boundaries.

Any errors or omissions remain the responsibility of the author.

---

## 16. References

* NIST FIPS 203: Module-Lattice-Based Key-Encapsulation Mechanism Standard  
  https://doi.org/10.6028/NIST.FIPS.203

* BIP-360: Pay-to-Tapscript-Hash (P2TSH)  
  https://github.com/bitcoin/bips/blob/master/bip-0360.mediawiki

* RFC 8439: ChaCha20 and Poly1305 for IETF Protocols  
  https://www.rfc-editor.org/rfc/rfc8439

* RFC 5869: HMAC-based Extract-and-Expand Key Derivation Function (HKDF)  
  https://www.rfc-editor.org/rfc/rfc5869

* RFC 8785: JSON Canonicalization Scheme (JCS)  
  https://www.rfc-editor.org/rfc/rfc8785

* Bitcoin Core:  
  https://github.com/bitcoin/bitcoin

---

# Annexes

## Annex A: SubmissionEvidence JSON Schema

Status: Normative

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["submission_id", "miner_id", "template_hash", "timestamp", "status", "signature"],
  "properties": {
    "submission_id": {
      "type": "string",
      "format": "uuid"
    },
    "miner_id": {
      "type": "string",
      "pattern": "^[0-9a-f]{64}$"
    },
    "template_hash": {
      "type": "string",
      "pattern": "^[0-9a-f]{64}$"
    },
    "timestamp": {
      "type": "string",
      "format": "date-time"
    },
    "status": {
      "type": "string",
      "enum": ["accepted", "rejected"]
    },
    "signature": {
      "type": "string"
    },
    "optional": {
      "type": "object",
      "properties": {
        "epoch_clock_hash": { "type": "string" },
        "inclusion_estimate": { "type": "integer" }
      }
    }
  }
}
```

### Illustrative SubmissionEvidence Example (Informative)

```json
{
  "submission_id": "6b9f1d2c-3e4a-4c88-9a9e-21b6a0e5f3a2",
  "miner_id": "a4f9c2e8c0a6c3b0e4f3a1b7c9d8e2f1a0b9c8d7e6f5a4b3c2d1e0f9a8b",
  "template_hash": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
  "timestamp": "2026-01-17T12:00:00Z",
  "status": "accepted",
  "optional": {
    "epoch_clock_hash": "0f3c9b1e7a5d4c2b9f8e6d1c0a7b5e3f2d4c6a8b9e1f3d5c7a9b0e2"
  },
  "signature": "<ed25519-signature-over-canonical-json>"
}
```

---

## Annex B: MinerKEMProfile JSON Schema

Status: Normative

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["miner_id", "kem_public_key", "valid_from", "valid_until", "signature"],
  "properties": {
    "miner_id": {
      "type": "string",
      "pattern": "^[0-9a-f]{64}$"
    },
    "key_id": {
      "type": "string",
      "description": "Optional identifier for the specific ML-KEM public key version (informative)"
    },
    "kem_public_key": {
      "type": "string"
    },
    "valid_from": {
      "type": "string",
      "format": "date-time"
    },
    "valid_until": {
      "type": "string",
      "format": "date-time"
    },
    "signature": {
      "type": "string"
    },
    "optional_metadata": {
      "type": "object",
      "properties": {
        "endpoint_url": { "type": "string", "format": "uri" },
        "fee_offer": { "type": "object" },
        "recommended_expiry_seconds": { "type": "integer" },
        "max_clock_skew_seconds": { "type": "integer" }
      }
    }
  }
}
```

### Illustrative MinerKEMProfile Example (Informative)

```json
{
  "miner_id": "a4f9c2e8c0a6c3b0e4f3a1b7c9d8e2f1a0b9c8d7e6f5a4b3c2d1e0f9a8b",
  "key_id": "kem-2026-01",
  "kem_public_key": "BASE64_ML_KEM_768_PUBLIC_KEY",
  "valid_from": "2026-01-01T00:00:00Z",
  "valid_until": "2026-03-01T00:00:00Z",
  "optional_metadata": {
    "endpoint_url": "https://miner.example/seal/submit",
    "recommended_expiry_seconds": 300,
    "max_clock_skew_seconds": 60
  },
  "signature": "<ed25519-signature-over-canonical-json>"
}
```


---

## Annex C: Execution State Transition Machine

Status: Normative

```
┌─────────┐
│ PENDING │
└────┬────┘
     │ (SubmissionEvidence verified)
     ▼
┌───────────┐     ┌──────────┐
│ SUBMITTED │────▶│ CONFIRMED│
└─────┬─────┘     └──────────┘
      │
      │ (failure detected)
      ▼
  ┌────────┐
  │ FAILED │
  └────┬───┘
       │ (explicit authorization)
       ▼
┌──────────────────┐
│ AUTHORIZED_PUBLIC│
└──────────────────┘
```

Allowed transitions:

* `PENDING → SUBMITTED`: Only after verified evidence
* `SUBMITTED → CONFIRMED`: Only after confirmation
* `SUBMITTED → FAILED`: On any failure
* `FAILED → AUTHORIZED_PUBLIC`: Only with explicit authorization

Prohibited:

* Direct `PENDING → CONFIRMED`
* Automatic `FAILED → SUBMITTED`
* Any transition bypassing `FAILED` state after failure

---

## Annex D: KeyMail Integration (Informative)

Status: Informative

KeyMail is an optional asynchronous encrypted transport for SEAL submissions.

### D.1 Overview

KeyMail provides:

* Cryptographic addressing (public key-based identities)
* Store-and-forward delivery
* End-to-end encryption
* Asynchronous message delivery

### D.2 Integration with SEAL

KeyMail as transport:

1. Wallet encrypts transaction using SEAL protocol
2. Wallet wraps encrypted SEAL payload in KeyMail message
3. Wallet sends KeyMail message to Submission Endpoint's KeyMail address
4. Submission Endpoint retrieves message, decrypts SEAL payload
5. Submission Endpoint validates and returns `SubmissionEvidence` via KeyMail reply

Important:

* KeyMail is a transport layer only
* SEAL encryption is independent of KeyMail encryption (defense in depth)
* KeyMail delivery does not constitute `SubmissionEvidence`

### D.3 KeyMail Addressing

Submission Endpoints MAY publish KeyMail addresses in `MinerKEMProfile` optional metadata:

```json
{
  "optional_metadata": {
    "keymail_address": "<public-key>@keymail.network"
  }
}
```

### D.4 Compatibility

KeyMail support is optional. SEAL operates correctly with or without KeyMail.

---

## Annex E: Relationship to PQSF (Informative)

Status: Informative

PQSF (Post-Quantum Security Framework) is a companion specification that defines canonical encoding and cryptographic discipline for post-quantum Bitcoin systems.

### E.1 SEAL's Use of PQSF

SEAL adopts PQSF principles where applicable:

* Canonical JSON encoding for signed objects
* Deterministic signature input construction
* Strict cryptographic parameter validation

### E.2 Independence

SEAL is independently implementable without PQSF.

PQSF provides additional rigor for implementations requiring formal cryptographic discipline.

---

## Annex F: Reference Implementation Notes

Status: Informative

---

### F.1 Wallet Implementation Notes

State persistence:

Wallets SHOULD persist execution state and SubmissionEvidence durably across restarts. On restart, wallets SHOULD default to a conservative posture and MUST NOT advance execution state without verified observation of confirmation or failure.

Ambiguity, loss of state, or observer unavailability SHOULD result in an execution freeze and transition to FAILED.

---

Mempool and chain observation:

Wallets SHOULD integrate with Bitcoin Core RPC or an equivalent trusted source to observe:

* confirmation status,
* public mempool visibility,
* blockchain reorganizations.

Unexpected public mempool exposure MUST be treated as terminal for the current execution attempt and surfaced immediately.

---

Expiry selection:

Wallets SHOULD keep expiry windows short relative to the intended execution policy to bound replay and delayed-delivery risk.

Wallets SHOULD include allowance for clock skew and transport latency between the wallet and the Submission Endpoint when setting `expires_at`.

A conservative wallet policy MAY add a small additional safety margin beyond any endpoint-advertised recommendations to account for worst-case transport delay and local clock uncertainty, while still keeping the total expiry window on the order of minutes to preserve strong replay protection.

Wallets targeting ultra-short expiry windows MAY choose tighter values and accept higher rejection risk due to clock skew or latency. This is a policy choice and does not affect SEAL correctness.

---

Clock skew observation:

Wallets MAY observe and record apparent clock skew between local time and Submission Endpoint time using diagnostic information returned in SubmissionEvidence or rejection responses.

If observed skew consistently exceeds endpoint-advertised limits or local policy thresholds, wallets MAY warn the user, adjust expiry selection, or recommend switching endpoints.

Clock skew observation is advisory only and has no execution semantics.

---

User interface behavior:

Wallets SHOULD clearly indicate when private relay execution is active.

Wallets SHOULD surface execution state transitions, failures, and freezes explicitly and distinguish private execution from public broadcast.

Wallets SHOULD require explicit confirmation before abandoning private execution and broadcasting publicly.

---

### F.2 Submission Endpoint Implementation Notes

Key management:

Submission Endpoints SHOULD store ML-KEM-768 private keys in secure storage such as an HSM or secure enclave where available.

Endpoints SHOULD implement automated key rotation and revocation procedures and retain audit logs of key usage.

---

Validation discipline:

Submission Endpoints SHOULD perform full decryption and validation of encrypted submissions before issuing SubmissionEvidence.

Endpoints SHOULD reject expired submissions prior to attempting decryption.

Replay protection MAY be implemented using submission identifiers or other bounded mechanisms.

---

Diagnostics and auditability:

Submission Endpoints MAY include diagnostic time information in SubmissionEvidence or rejection responses to assist with clock skew analysis and operational debugging.

Diagnostic fields are advisory only and MUST NOT be interpreted as inclusion intent, ordering guarantees, or execution promises.

---

Operational posture:

Submission Endpoints MAY delay, withhold, or refuse transactions based on local policy without violating SEAL.

Endpoints SHOULD avoid issuing ambiguous or misleading evidence and SHOULD retain SubmissionEvidence for a policy-defined period to support audit or dispute analysis.

---

### F.3 Transport Examples (Informative)

#### Encrypted Submission (HTTP)

```http
POST /seal/submit HTTP/1.1
Content-Type: application/json

{
  "miner_id": "a4f9c2e8c0a6c3b0e4f3a1b7c9d8e2f1a0b9c8d7e6f5a4b3c2d1e0f9a8b",
  "key_id": "kem-2026-01",
  "submission_id": "6b9f1d2c-3e4a-4c88-9a9e-21b6a0e5f3a2",
  "template_hash": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
  "expires_at": "2026-01-31T00:05:00Z",
  "kem_ciphertext": "BASE64_KEM_CIPHERTEXT",
  "nonce": "BASE64_XCHACHA_NONCE",
  "ciphertext": "BASE64_ENCRYPTED_TRANSACTION",
  "tag": "BASE64_AEAD_TAG"
}
```

#### Accepted Response

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "submission_id": "6b9f1d2c-3e4a-4c88-9a9e-21b6a0e5f3a2",
  "miner_id": "a4f9c2e8c0a6c3b0e4f3a1b7c9d8e2f1a0b9c8d7e6f5a4b3c2d1e0f9a8b",
  "template_hash": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
  "timestamp": "2026-01-17T12:00:00Z",
  "status": "accepted",
  "signature": "<ed25519-signature-over-canonical-json>"
}
```

#### Rejection Response

```http
HTTP/1.1 400 Bad Request
Content-Type: application/json

{
  "status": "rejected",
  "rejection_code": "TEMPLATE_HASH_MISMATCH",
  "timestamp": "2026-01-17T12:00:02Z",
  "details": "Template hash mismatch after decryption"
}
```
---

## Annex G: Future Work and Research Directions

Status: Informative

This annex documents potential areas of future research related to execution-layer security and post-quantum Bitcoin systems. Nothing in this annex affects SEAL correctness, conformance, or current security guarantees.

---

### G.1 Scope Boundary

SEAL is intentionally conservative. It defines a minimal execution-layer mitigation for execution-window exposure and avoids pre-empting consensus-layer, custody-layer, or policy-layer design choices.

All items listed here are explicitly non-binding and exploratory.

---

### G.2 Potential Research Areas

Future work MAY explore, without implying requirement or endorsement:

* transport-layer privacy improvements for encrypted submission,
* improved correlation resistance for multi-endpoint execution,
* richer audit and accountability tooling built on SubmissionEvidence,
* reputation systems external to SEAL for endpoint evaluation,
* alternative post-quantum cryptographic primitives as standards evolve.

Any such work MUST preserve SEAL’s execution discipline, explicit authorization boundaries, and failure semantics.

---

### G.3 Consensus-Layer Post-Quantum Migration

Long-term cryptographic resilience for Bitcoin ultimately requires consensus-layer upgrades, such as post-quantum signature schemes or new Script semantics.

SEAL does not depend on, accelerate, or constrain such changes. It is designed to remain compatible with future consensus evolution without creating transitional coupling.

---

### G.4 Non-Implication Statement

The presence of this annex MUST NOT be interpreted as:

* a roadmap commitment,
* a promise of future protocol changes,
* an expansion of SEAL’s security claims,
* or a weakening of the current conservative posture.

SEAL’s security guarantees are defined solely by the normative sections of this specification.

---

## Annex H: Naming Consistency and Terminology Discipline

Status: Informative

---

### Prohibited or Discouraged Terms

The following terms MUST NOT be used, as they imply guarantees or properties SEAL does not provide:

* private mempool
* secure broadcast
* encrypted broadcast
* quantum-safe transaction
* miner guarantee
* inclusion guarantee
* censorship resistance via SEAL
* temporary fix
* stopgap mechanism

Use of these terms constitutes misrepresentation of SEAL’s security model.

---

### Required Terminology

The following terms MUST be used with the meanings defined in this specification:

* SEAL
  The execution-layer protocol defined here.

* Submission Endpoint
  An identity-bound execution-layer interface that accepts encrypted submissions, validates transactions, and may attempt inclusion. Endpoint identity does not imply control of physical hashpower.

* MinerKEMProfile
  The signed object published by a Submission Endpoint containing ML-KEM public key material, validity windows, and identity binding.

* SubmissionEvidence
  A cryptographically signed attestation that a Submission Endpoint successfully decrypted and validated an encrypted submission. SubmissionEvidence proves submission and validation only.

* Execution Window
  The interval between transaction signing and on-chain confirmation during which signatures would otherwise be publicly observable.

* Execution Freeze
  A halt in execution progression triggered by failure, exposure, or ambiguity, pending explicit authorization.

* Abandonment of Private Execution
  An explicit recovery action authorizing public broadcast and acknowledging execution-window exposure.

---

### Terminology Boundaries

* Miner refers exclusively to block production.
* Submission Endpoint refers exclusively to execution-layer interaction.
* Evidence must always be qualified as SubmissionEvidence.
* Private Relay refers to encrypted execution-layer delivery, not to any mempool-like structure.

Descriptions that blur these boundaries risk incorrect threat assumptions and unsafe implementations.

---

### Conformance Note

Implementations and documentation that materially deviate from this terminology risk over-claiming security properties and SHOULD be considered non-conformant in spirit, even if technically functional.

---

## Annex I: Relationship to Bitcoin Pre-Contracts (BPC)

Status: Informative
Audience: Protocol designers, reviewers
Normative dependency: None

---

### I.1 Shared Core Invariant

SEAL and Bitcoin Pre-Contracts share a foundational invariant:

> A broadcast-valid transaction MUST NOT be publicly visible prior to intended execution.

This invariant is enforced differently in each system.

---

### I.2 SEAL vs BPC — Scope Distinction

#### SEAL

* operates after transaction construction
* assumes transaction is already valid and signed
* defers public visibility
* enforces execution discipline
* preserves unconditional user broadcast authority

#### Bitcoin Pre-Contracts (BPC)

* operate before transaction construction
* enforce authorization and refusal predicates
* may prevent transaction creation entirely
* introduce additional authority layers

---

### I.3 Complementary Use

SEAL and BPC can be composed:

* BPC controls whether a transaction may be constructed
* SEAL controls how it is executed

Neither system subsumes the other.

---

### I.4 Censorship Resistance

SEAL preserves Bitcoin's censorship resistance by:

* never removing the ability to broadcast publicly
* requiring explicit authorization for abandonment of private execution
* avoiding miner coordination or enforcement

Any implementation that removes or disables user-initiated public broadcast is non-conformant.

---

### I.5 Summary

SEAL is a minimal, execution-layer protocol that embodies the "defer exposure" principle without introducing custody authority, refusal logic, or consensus dependencies.

It can coexist with, but does not replace, Bitcoin Pre-Contracts.

---

### Annex J: Related and Linked Work

Status: Informative  
Audience: Researchers, reviewers, implementers  
Normative dependency: None

This annex provides attribution, architectural context, and pointers to related
work authored or maintained by the SEAL author. None of the systems listed here
are required for SEAL correctness, conformance, or interoperability.

Implementations MUST NOT assume the presence, availability, or behavior of any
system referenced in this annex.

---

### Execution and Observation Context

* ZEB (Zero-Exposure Broadcasting)  
  Formal execution observation, restart safety, and failure surfacing semantics.  
  https://github.com/rosieRRRRR/ZEB

* bitcoin-quantum-spend  
  Research and modeling of quantum threats to Bitcoin signature exposure during execution.  
  https://github.com/rosieRRRRR/bitcoin-quantum-spend

---

### Post-Quantum Security Frameworks

* PQSF (Post-Quantum Security Framework)  
  Canonical cryptographic discipline and cross-layer post-quantum design.  
  https://github.com/rosieRRRRR/pqsf

* PQSEC  
  Threat modeling and security analysis for post-quantum Bitcoin systems.  
  https://github.com/rosieRRRRR/PQSEC

* PQ  
  High-level mapping of post-quantum risks and mitigations across Bitcoin layers.  
  https://github.com/rosieRRRRR/pq

---

### Execution Gating and Authorization

* PQEH  
  Execution-gated transaction construction and explicit authorization boundaries.  
  https://github.com/rosieRRRRR/PQEH

* Bitcoin Pre-Contracts (BPC)  
  Authorization-before-construction semantics for Bitcoin transactions.  
  https://github.com/rosieRRRRR/BPC

---

### Custody and Lifecycle Control

* PQHD  
  Post-quantum hierarchical custody and quorum authorization models.  
  https://github.com/rosieRRRRR/pqhd

---

### Runtime Integrity and AI Governance

* PQVL (Post-Quantum Verification Layer)  
  Runtime integrity verification and attestation frameworks.  
  https://github.com/rosieRRRRR/pqvl

* PQAI  
  Deterministic governance and integrity enforcement for AI systems.  
  https://github.com/rosieRRRRR/pqai

* Neural Lock  
  Experimental work on intent attestation and human authorization signals.  
  https://github.com/rosieRRRRR/neural-lock

---

### Time and Execution Anchoring

* Epoch Clock  
  Tamper-evident, externally verifiable time commitments for high-accountability systems.  
  https://github.com/rosieRRRRR/epoch-clock

---

### Historical Precursors

* SEAL-360  
  Early exploration of execution-layer exposure reduction. Superseded by this specification.  
  https://github.com/rosieRRRRR/SEAL-360
