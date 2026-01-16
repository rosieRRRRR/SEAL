# Seal-360 — Execution-Layer Mitigation for BIP-360

**Encrypted Direct-to-Miner Execution for P2TSH**

* **Specification Version:** 1.0.0
* **Status:** Draft - Specification complete, seeking public review.
* **Date:** January 2026
* **Author:** rosiea
* **Contact:** PQRosie@proton.me
* **Licence:** Apache License 2.0 — Copyright 2026 rosiea

---

## Abstract

Seal-360 is a voluntary execution-layer overlay that enables encrypted,
direct-to-miner submission of fully signed Bitcoin transactions, reducing public
mempool exposure prior to confirmation when private relay succeeds.

BIP-360 Pay-to-Tapscript-Hash (P2TSH) hardens outputs by removing Taproot
key-path spending but does not prevent signatures from being exposed once a
transaction is broadcast to the public mempool. Seal-360 addresses this
execution-window exposure by encrypting signed transactions and submitting them
directly to miners using ML-KEM-768.

When private relay succeeds, transaction signatures are not publicly observable
prior to confirmation. When private relay fails, public broadcast remains
available only through an Explicit Public Broadcast Authorization Event, after
which standard Bitcoin execution semantics apply and execution is quantum-unsafe.

Seal-360 does not introduce additional changes to Bitcoin consensus, Script semantics, custody models, or transaction validation rules beyond those required by BIP-360.

---

# Executive Summary (Non-Normative)

**Encrypted Direct-to-Miner Execution for Post-Quantum Bitcoin Security**

---

## Problem Statement

BIP-360 (Pay-to-Tapscript-Hash) hardens Bitcoin outputs against quantum attacks by removing Taproot key-path spending. However, it does not address a critical remaining risk: **the execution window between transaction signing and on-chain confirmation**.

BIP-360 is a consensus change that hardens outputs at the Script level but does
not alter transaction construction, broadcast, mempool behavior, or execution
semantics.

Under standard Bitcoin execution, signed transactions are broadcast to the public mempool, where signatures become globally observable for approximately one block interval. A quantum-capable adversary can exploit this window to extract signatures, recover private keys, and replace transactions before confirmation.

**BIP-360 protects outputs. Seal-360 protects execution.**

---

## Overview of the Solution

Seal-360 is an execution-layer protocol that eliminates public mempool exposure during transaction execution by enabling **encrypted, direct-to-miner submission** of fully signed Bitcoin transactions.

Instead of immediate public broadcast, wallets encrypt the signed transaction under a participating miner’s post-quantum public key and submit it privately. The transaction becomes publicly visible only after confirmation.

```

WITHOUT SEAL-360:
Sign → Broadcast → Public Mempool → Confirm

WITH SEAL-360:
Sign → Encrypt → Direct to Miner → Confirm

```

Seal-360 operates at the execution layer and does not introduce additional consensus, Script, or transaction validity changes beyond those required by BIP-360.

---

## Security Properties

### Provided (While Private Relay Succeeds)

- No public mempool exposure prior to confirmation
- Post-quantum confidentiality during submission (ML-KEM-768 + AEAD)
- Transaction integrity enforced via cryptographic template hash
- Explicit, user-controlled transition to public broadcast when required

### Not Provided

- Miner inclusion or timeliness guarantees
- Anonymity or metadata privacy
- Protection after explicit public broadcast authorization
- On-chain post-quantum signatures

Seal-360 reduces the execution-time observer set but does not eliminate miner visibility.

---

## Trust and Control Model

**Miners are not trusted.**  
They gain temporary visibility into the transaction but cannot modify it, create alternative spends, or force execution outcomes.

**Users retain sovereignty.**  
If private relay fails, execution state is surfaced and the user (or a pre-armed policy) explicitly chooses whether to retry, wait, abort, or authorize public broadcast. Public broadcast is never triggered automatically.

---

## Deployment Characteristics

**Wallets**
- Construct standard Bitcoin transactions
- Encrypt signed transactions for private relay
- Enforce bounded retry and timeout behavior
- Require explicit authorization before any public broadcast

**Miners**
- Publish post-quantum encryption keys
- Accept encrypted submissions
- Verify template hash integrity
- Process transactions under standard Bitcoin consensus rules

**Bitcoin nodes**
- Require no changes beyond those required to support BIP-360

Seal-360 provides incremental utility from a single participating miner and does not require ecosystem-wide adoption.

---

## Relationship to BIP-360

Seal-360 composes cleanly with BIP-360:

| Layer | Protection | Mechanism |
|------|-----------|-----------|
| Output | Quantum-safe outputs | P2TSH (BIP-360) |
| Execution | Reduced exposure | Encrypted direct submission |
| Combined | End-to-end hardening | BIP-360 + execution-layer mitigation |

Without Seal-360, BIP-360 outputs remain exposed during execution.

---

## Intended Use

Seal-360 is intended for:
- High-value settlement and treasury operations
- Quantum-sensitive execution environments
- Institutional or OTC transactions

It is not intended to replace standard Bitcoin broadcast for routine consumer payments.

---

## Summary

Seal-360 removes the execution-window exposure that BIP-360 leaves unresolved by deferring public broadcast and enabling encrypted, direct-to-miner execution.

- When private relay succeeds: no public mempool exposure
- When it fails: users explicitly authorize next steps
- Always: Bitcoin’s consensus model and censorship resistance remain intact

Seal-360 is a deployable, opt-in execution-layer mitigation that strengthens Bitcoin’s post-quantum security posture without altering the protocol itself.

---

## Index

0. [Scope](#0-scope)
1. [Terminology](#1-terminology)
2. [Design Goal](#2-design-goal)
3. [Execution Model Clarification (Normative)](#3-execution-model-clarification-normative)

4. [Why This Matters](#4-why-this-matters)
   4.1 [Execution-Window Exposure (Without Seal-360)](#41-execution-window-exposure-without-seal-360)
   4.2 [Eliminating the Execution Window (With Seal-360)](#42-eliminating-the-execution-window-with-seal-360)
   4.3 [Security Boundary Clarification](#43-security-boundary-clarification)
   4.4 [Liveness vs. Quantum Safety Tradeoff (Informative)](#44-liveness-vs-quantum-safety-tradeoff-informative)
   4.5 [Default Behavior Guidance (Informative)](#45-default-behavior-guidance-informative)

5. [Non-Goals](#5-non-goals)

6. [System Assumptions](#6-system-assumptions)
   6.1 [Wallet](#wallet)
   6.2 [Miner](#miner)
   6.3 [Timestamp Format (Normative)](#timestamp-format-normative)

7. [Threat Model](#7-threat-model)
   7.1 [In-Scope Threats](#71-in-scope-threats)
   7.2 [Out-of-Scope Threats](#72-out-of-scope-threats)
   7.3 [Miner Visibility Boundaries](#73-miner-visibility-boundaries)
   7.4 [Economic Incentives, Abuse, and Correlation Risk](#74-economic-incentives-abuse-and-correlation-risk)
       7.4.1 [Miner Incentives](#741-miner-incentives)
       7.4.2 [Adversarial Miner Behavior](#742-adversarial-miner-behavior)
       7.4.3 [Miner Adoption and Deployment Bootstrap (Informative)](#743-miner-adoption-and-deployment-bootstrap-informative)
       7.4.4 [Wallet Fee Discipline (Recommended)](#744-wallet-fee-discipline-recommended)
       7.4.5 [Failure Handling and Retry Guidance (Informative)](#745-failure-handling-and-retry-guidance-informative)
   7.5 [Visibility Guarantees](#75-visibility-guarantees)
   7.6 [Residual Quantum Risk](#76-residual-quantum-risk)
   7.7 [Design Rationale and Common Objections (Non-Normative)](#77-design-rationale-and-common-objections-non-normative)
       7.7.1 [Rationale for User-Controlled Execution (Informative)](#771-rationale-for-user-controlled-execution-informative)

8. [Protocol Overview](#8-protocol-overview)

9. [Protocol Operation](#9-protocol-operation)
   9.1 [Miner Discovery](#91-miner-discovery)
       9.1.1 [Provider Metadata and Endpoint Validation](#911-provider-metadata-and-endpoint-validation)
       9.1.2 [Public Key Authenticity](#912-public-key-authenticity)
   9.2 [Transaction Construction](#92-transaction-construction)
   9.3 [Template Hashing (Normative)](#93-template-hashing-normative)
       9.3.1 [Canonical Hash Input (Normative)](#931-canonical-hash-input-normative)
       9.3.2 [Serialization Requirements (Normative)](#932-serialization-requirements-normative)
       9.3.3 [Hash Function (Normative)](#933-hash-function-normative)
       9.3.4 [Binding Properties (Normative)](#934-binding-properties-normative)
       9.3.5 [Miner Verification Rule (Normative)](#935-miner-verification-rule-normative)
       9.3.6 [Spend Form Constraint (Normative)](#936-spend-form-constraint-normative)
   9.4 [Miner Offer Object (Informational)](#94-miner-offer-object-informational)
   9.5 [Transaction Encryption](#95-transaction-encryption)
       9.5.1 [Recommended Encryption Profile (Normative)](#951-recommended-encryption-profile-normative)
       9.5.2 [Transitional Encryption Profile](#952-transitional-encryption-profile)
       9.5.3 [Canonical AAD Encoding (Normative)](#953-canonical-aad-encoding-normative)
   9.6 [Encrypt-Before-Transport Semantics (Normative)](#96-encrypt-before-transport-semantics-normative)
       9.6.1 [HTTPS/TLS as Authenticated Carrier (Recommended)](#961-httpstls-as-authenticated-carrier-recommended)
       9.6.2 [Optional Binary Framing (Framing Only)](#962-optional-binary-framing-framing-only)
       9.6.3 [Carrier Selection and Failure Handling](#963-carrier-selection-and-failure-handling)
       9.6.4 [Carrier Security Scope](#964-carrier-security-scope)
   9.7 [Miner Processing (Normative)](#97-miner-processing-normative)
   9.8 [Public Broadcast Authorization](#98-public-broadcast-authorization)
       9.8.1 [Correlation Risk](#981-correlation-risk)
       9.8.2 [Correlation Downgrade Guard](#982-correlation-downgrade-guard)
   9.9 [Post-Execution Output Anchoring](#99-post-execution-output-anchoring)
   9.10 [Replace-By-Fee (RBF)](#910-replace-by-fee-rbf)
       9.10.1 [RBF Semantics Under Private Relay (Normative)](#9101-rbf-semantics-under-private-relay-normative)
   9.11 [Normative Scope](#911-normative-scope)

10. [Security Properties](#10-security-properties)
11. [Failure Modes](#11-failure-modes)

12. [Deployment Requirements](#12-deployment-requirements)
    12.1 [Wallet Requirements](#121-wallet-requirements)
    12.2 [Miner Requirements](#122-miner-requirements)
    12.3 [Bitcoin Node Requirements](#123-bitcoin-node-requirements)
    12.4 [Optional Infrastructure](#124-optional-infrastructure)
    12.5 [Backward Compatibility](#125-backward-compatibility)

13. [Summary](#13-summary)
14. [Conformance Statement](#14-conformance-statement)
15. [Normative References](#15-normative-references)
16. [Informative References](#16-informative-references)

### Annexes

A. [Optional Hardening Extensions](#annex-a-optional-hardening-extensions)
B. [Comparative Analysis](#annex-b-comparative-analysis)
C. [Test Vectors and Conformance Tests](#annex-c-test-vectors-and-conformance-tests-reference)
D. [Deployment Scenarios](#annex-d-deployment-scenarios)
E. [Frequently Asked Questions](#annex-e-frequently-asked-questions)
F. [Wallet UX Guidelines and Miner Selection Interface (Informative)](#annex-f-wallet-ux-guidelines-and-miner-selection-interface-informative)
G. [Terminology](#annex-g-terminology)
H. [Miner Discovery Protocol (Normative)](#annex-h-miner-discovery-protocol-normative)
I. [Optional Stack Integration (Informative)](#annex-i-optional-stack-integration-informative)
J. [Protocol Flow Diagrams](#annex-j-protocol-flow-diagrams)
K. [Optional PQHD Keymail Transport (Informative)](#annex-k-optional-pqhd-keymail-transport-informative)
L. [Minimum Viable Implementation (Informative)](#annex-l-minimum-viable-implementation-informative)
M. [Reference Code, Visual Summary, and Naming (Informative)](#annex-m-reference-code-visual-summary-and-naming-informative)
N. [Relationship to Bitcoin Pre-Contracts (Informative)](#annex-n-relationship-to-bitcoin-pre-contracts-informative)

---

## 0. Scope

This document specifies **Seal-360**, an execution-layer mitigation whose sole purpose is to remove public mempool exposure during transaction execution.

Seal-360:

* addresses only the broadcast-to-confirmation window,
* introduces no custody semantics,
* introduces no authorization rules,
* introduces no additional consensus or Script changes beyond those required by BIP-360.

---

## 1. Terminology

This specification uses the following terms with precise meanings.

**Seal-360**  
The execution-layer protocol defined in this document, specifying cryptographic binding, execution semantics, authorization rules, and failure handling for encrypted direct-to-miner transaction submission.

**Private relay**  
The Seal-360 execution mode in which a signed transaction is delivered privately to a miner prior to confirmation. Private relay includes miner discovery, transaction construction and signing, encrypted submission, miner-side verification, and explicit execution outcome handling. Private relay explicitly excludes automatic public broadcast.

**Encrypt-Before-Transport (EBT)**  
A security invariant requiring that the signed transaction payload is
cryptographically encrypted prior to entering any transport, framing, routing,
or delivery mechanism. EBT defines payload confidentiality only and does not
define transport authentication, routing, retry, ordering, or delivery
semantics.

**Encrypted Direct-to-Miner Submission** (or **Encrypted Submission**)  
The protocol action of encrypting a fully signed Bitcoin transaction under a miner’s ML-KEM-768 public key and transmitting it directly to that miner for potential block inclusion. Encrypted Submission is a mechanism within private relay and does not imply inclusion guarantees.

**Public broadcast**  
Submission of a fully signed Bitcoin transaction to the public peer-to-peer network and mempool. Public broadcast is always available to the user or policy engine but is never triggered automatically by the Seal-360 protocol. When authorized, public broadcast re-enters the standard Bitcoin threat model and is considered quantum-unsafe during the execution window.

**Execution window**  
The interval between transaction signing and blockchain confirmation during which transaction signatures may become observable. In standard Bitcoin execution, this window includes public mempool exposure. Seal-360 eliminates public mempool exposure during this window while private relay succeeds.

This section defines terminology required to interpret the core Seal-360
protocol. Extended, cross-stack, or conceptual definitions are provided in
Annex G and are informative unless explicitly stated otherwise.

---

## 2. Design Goal

**Reduce public mempool exposure during transaction execution.**

Under standard Bitcoin execution, signed transactions are exposed in the public mempool for approximately one block interval.

During this window:

* signatures are publicly observable,
* replacement attacks are possible,
* post-quantum adversaries can recover private keys.

Seal-360 reduces this exposure by enabling wallets to submit signed transactions directly to miners using post-quantum payload encryption.

Seal-360 guarantees no public mempool exposure only while private relay is active and successful. If private relay fails and public broadcast is explicitly authorized, standard Bitcoin exposure semantics apply.

## 3. Execution Model Clarification (Normative)

Seal-360 defines an execution-layer delivery mechanism for signed Bitcoin transactions.

Public broadcast remains always available through explicit user authorization but is never triggered automatically by the protocol.

This preserves Bitcoin’s censorship resistance while eliminating automatic quantum exposure during the execution window.

Seal-360 modifies **how** a transaction may be delivered prior to confirmation, not **whether** it may enter Bitcoin’s normal consensus process.

Seal-360 defines no new validation rules and introduces no new script forms; it specifies only an optional delivery mechanism for already-valid BIP-360 transactions and therefore does not constitute a Bitcoin consensus change.


---

## 4. Why This Matters

### 4.1 Execution-Window Exposure (Without Seal-360)

**Assumptions:**

* Funds are held in a BIP-360 Pay-to-Tapscript-Hash (P2TSH) output.
* A cryptographically relevant quantum computer exists.
* Standard Bitcoin broadcast and relay rules apply.

In standard Bitcoin operation, a wallet constructs and signs a transaction using
secp256k1 signatures and immediately broadcasts it to the public peer-to-peer
network.

Once broadcast, the transaction enters the public mempool and becomes globally
visible prior to confirmation.

**Typical execution timeline (standard Bitcoin):**

```
T+0        Wallet constructs and signs transaction (secp256k1)
T+0        Transaction is broadcast to the public mempool
↓
           [Transaction is visible to all nodes, observers, and adversaries]
           [Signatures are public and authoritative]
↓
T+~10–60s  Adversary extracts signature(s) from the mempool
T+~1–2 min Quantum adversary attempts private key recovery
T+~2–5 min Adversary constructs a competing spend
T+~5–10 min Adversary broadcasts higher-fee replacement (RBF or CPFP)
↓
T+~10 min  Adversary’s transaction confirms
↓
Outcome: Funds stolen. Original transaction never confirms.
```

This vulnerability exists **after signing but before confirmation**.

BIP-360 significantly hardens the output script by removing the Taproot key-path
spend, but it provides no protection once a signed spending transaction is
broadcast to the public mempool.

As a result, the execution window remains fully exposed. Signature disclosure
during this window is sufficient for a quantum-capable adversary to compromise
funds in real time.

---

### 4.2 Eliminating the Execution Window (With Seal-360)

Seal-360 replaces immediate public broadcast with encrypted,
direct-to-miner submission.

Prior to signing, the wallet selects a participating miner that has published:

* a valid ML-KEM-768 public encryption key,
* current fee terms, and
* an offer validity window.

The wallet then constructs and signs the transaction normally, encrypts the
signed transaction under the miner’s post-quantum public key, and submits the
encrypted payload directly to the miner. The transaction never enters the public
mempool during execution.

**Execution timeline with Seal-360 (successful private relay):**

```
T+0        Wallet selects miner offer (fee terms, validity window)
T+0        Wallet constructs and signs transaction (secp256k1)
T+0        Transaction is encrypted using ML-KEM-768
T+0        Encrypted Submission sent directly to selected miner
↓
           [No public mempool broadcast occurs]
           [Signatures remain encrypted and non-observable]
↓
T+~10 min  Miner decrypts, verifies template_hash integrity,
           and includes transaction in a block
↓
           Transaction becomes public only after confirmation
↓
Outcome: Execution-window exposure eliminated.
```

When private relay succeeds:

* Transaction signatures are never exposed in the public mempool.
* The quantum-vulnerable execution window is reduced to zero.
* The transaction becomes observable to the network only after confirmation.

Bitcoin consensus rules, transaction validity, fee logic, and signature schemes remain unchanged beyond those defined by BIP-360.

**Important distinction:**

Seal-360 does not alter how transactions are signed or validated. It changes
*when* and *to whom* a signed transaction becomes visible.

This shifts execution-time visibility from the entire network to a single,
temporary miner endpoint, providing a clean execution-layer mitigation that
complements BIP-360’s output hardening without introducing additional consensus changes.

---

### 4.3 Security Boundary Clarification

Seal-360 addresses **execution-layer exposure only**.

It does not:

* provide on-chain post-quantum signatures,
* guarantee miner inclusion,
* prevent miner censorship,
* eliminate risk after public broadcast.

When public broadcast is explicitly authorized, standard Bitcoin threat assumptions apply immediately.

Seal-360 is a strict, additive mitigation that removes a critical execution-window attack surface while preserving Bitcoin’s existing security and liveness properties.

---

## 4.4 Liveness vs. Quantum Safety Tradeoff (Informative)

Seal-360 cannot simultaneously guarantee both transaction liveness and quantum-safe execution.

Seal-360 provides the execution mechanism. Policy determines behavior.

**Execution Modes:**

**Quantum-Safety–Priority Mode**

* No automatic public broadcast
* Private relay attempts may be retried
* Transaction may remain unconfirmed if all miners refuse
* User retains the ability to manually authorize public broadcast, accepting quantum exposure

**Liveness–Priority Mode**

* Public broadcast MAY be authorized after timeout **only** via an Explicit Public Broadcast Authorization Event
* Transaction propagates via standard Bitcoin relay
* Quantum protection is lost once public broadcast occurs
* Execution semantics become equivalent to standard Bitcoin

**Implementation Guidance:**

* Implementations MAY support both modes.
* Implementations SHOULD clearly disclose the security and liveness implications of each mode.
* Implementations MAY select defaults appropriate to their threat model and user base.
* Seal-360 does not mandate a default mode at the protocol level.

---

### 4.5 Default Behavior Guidance (Informative)

Seal-360 specifies execution mechanisms, not policy defaults.

No default behavior is mandated by this specification. Wallets and operators are
responsible for selecting defaults appropriate to their deployment context,
threat model, and user expectations.

The following guidance is non-normative:

* Implementations MAY default to attempting private relay for transactions
  spending BIP-360 (P2TSH) outputs, where execution-window exposure reduction is
  the primary motivation.
* Implementations MAY default to standard public broadcast behavior for
  non-BIP-360 outputs or environments where confirmation liveness outweighs
  execution-window risk.
* Implementations MUST require explicit user or pre-armed policy authorization
  before any public broadcast occurs following a private relay attempt.
* Implementations MUST provide a clear mechanism for users or operators to
  override defaults through explicit configuration or policy.

No timeout, retry exhaustion, transport failure, or discovery failure constitutes
authorization to broadcast by itself.

Seal-360 does not define, recommend, or imply a universal default execution mode.

---

## 5. Non-Goals

Seal-360 explicitly does not attempt to:

* redefine wallet custody,
* change Bitcoin signing rules,
* enforce authorization policies,
* guarantee miner inclusion,
* replace the fee market,
* prevent miner censorship,
* claim complete quantum safety.

Seal-360 is **strictly an execution-layer mitigation**.

---

## 6. System Assumptions

### Wallet

* Standard Bitcoin transaction construction
* secp256k1 signing
* PSBT v2 or raw transaction support
* No execution gating beyond deferred broadcast

### Miner

* Standard block construction
* Voluntary support for private relay
* Publication of a public encryption key
* No changes to block or transaction validation rules beyond those defined by BIP-360

### Timestamp Format (Normative)

All timestamps used in Seal-360 MUST be expressed as RFC 3339 timestamps in UTC.

- Format: `YYYY-MM-DDTHH:MM:SSZ`
- Seconds precision is REQUIRED.
- Fractional seconds MAY be included.
- Local timezones, offsets, and ambiguous ISO 8601 variants are NOT permitted.

Any timestamp that does not conform to RFC 3339 UTC format MUST be rejected.


---

## 7. Threat Model

Seal-360 is an execution-layer mitigation. It does not modify consensus, Script, or signature semantics beyond those defined by BIP-360.

Its security objective is limited to eliminating **public mempool visibility prior to confirmation**.

### 7.1 In-Scope Threats

* Public mempool observation
* Broadcast-time signature extraction
* Post-quantum key recovery during execution
* Replacement attacks exploiting recovered keys
* Network-level transaction gossip analysis

Seal-360 mitigates these threats by ensuring that **no signed transaction is publicly broadcast prior to confirmation**, except when public broadcast is explicitly authorized.

---

### 7.2 Out-of-Scope Threats

* Miner censorship or refusal
* Miner fee manipulation or reordering
* Wallet key compromise
* Side-channel attacks
* Cryptographic implementation flaws
* Long-term on-chain key exposure

---

### 7.3 Miner Visibility Boundaries

Miners are not trusted.

During private relay, the selected miner gains exclusive, temporary visibility
into the signed transaction prior to confirmation.

As a result, a miner MAY:

* observe full transaction contents during the private relay window,
* refuse inclusion without explanation,
* delay inclusion to influence timing or fees,
* leak transaction data to third parties,
* extract informational, economic, or ordering advantage from exclusive
  visibility.

A miner MUST NOT, without detection:

* modify transaction bytes after signing, or
* create an alternative valid spend.

Any modification to transaction bytes invalidates the `template_hash` and MUST
result in rejection.

Seal-360 intentionally trades global public mempool exposure for concentrated
miner visibility during execution. This reduces the observer set but does not
eliminate surveillance, economic leverage, or correlation risk.

Miner visibility does not imply execution authority, custody, or control over
broadcast. All execution outcomes remain bounded by wallet-defined timeouts and
explicit user or policy authorization.

---

## 7.4 Economic Incentives, Abuse, and Correlation Risk

Seal-360 is voluntary. Miners are not trusted and may refuse any submission for any reason. This section defines the economic incentives for participation, the abuse surface, and the residual risks that wallets MUST account for.

### 7.4.1 Miner Incentives

Miners MAY support private relay because it is economically compatible with existing incentives:

* **Fee capture**
  Miners receive standard transaction fees and MAY charge a premium for accepting Encrypted Submissions.

* **Operational efficiency**
  Direct submission reduces gossip overhead and malformed transaction handling relative to public mempool ingestion.

* **Service differentiation**
  Pools MAY offer private relay as a premium service for latency-sensitive or high-value flows without requiring additional consensus changes or coordination with other miners.

Seal-360 does not assume altruism. Participation is optional and market-driven.

---

### 7.4.2 Adversarial Miner Behavior

Wallets MUST assume that a selected miner may behave adversarially.

During private relay, a miner MAY:

* Refuse to accept an Encrypted Submission.
* Delay inclusion until a wallet-defined timeout is reached.
* Leak a decrypted transaction prior to confirmation.
* Use selective refusal, delay, or reliability degradation to exert economic
  pressure, including fee escalation or premium extraction.
* Exploit exclusive pre-confirmation visibility to gain ordering or
  miner-extractable value (MEV), where applicable.

A miner MUST NOT, without detection:

* Modify transaction bytes after signing. Any such modification invalidates the
  template hash and MUST result in rejection.
* Construct an alternative valid spend. The miner does not possess signing keys.

Adversarial miner behavior can degrade privacy, delay execution, and introduce
economic coercion or ordering risk. It does not grant execution authority,
custody, or unilateral control over transaction outcome.

Containment properties:

* Miner misbehavior is bounded by wallet-defined retry limits and timeouts.
* Private relay failure MUST NOT trigger automatic public broadcast.
* Execution state MUST be surfaced for explicit user or policy decision.
* Users or policies MAY choose to retry, wait, authorize public broadcast, or
  abort execution.

Seal-360 treats miner misbehavior as an execution failure, not a protocol
violation.

### 7.4.3 Miner Adoption and Deployment Bootstrap (Informative)

Seal-360 does not require broad miner adoption to be useful. Initial deployment requires only a small number of participating miners and wallet implementations.

#### Initial Adoption Path

Early deployment is expected to occur through miners and pools that already operate:

* direct-to-miner submission APIs,
* transaction accelerator or priority inclusion services,
* private relay or whitelisted peer infrastructure,
* institutional OTC or settlement channels.

For these operators, Seal-360 requires only:

* accepting encrypted transaction payloads,
* verifying `template_hash`,
* feeding accepted transactions into existing internal mempool or block-template pipelines.

No additional consensus changes, inter-miner coordination, or policy commitments are required beyond those already introduced by BIP-360. Adoption is market-driven and requires no coordination beyond individual miner decisions.

#### Incremental Utility Model

Seal-360 utility scales incrementally:

* One participating miner provides immediate value for high-value transactions.
* Multiple miners reduce correlation and refusal risk.
* Widespread adoption is not a prerequisite for correctness or liveness.

Wallets MAY attempt private relay opportunistically and MUST preserve the ability for user-authorized public broadcast to maintain liveness.

#### Risk-Free Miner Participation

Miners MAY:

* rate-limit or refuse submissions,
* restrict access to specific counterparties,
* offer private relay only for certain fee tiers,
* disable the service at any time.

There is no lock-in and no obligation to include transactions.

#### 7.4.4 Wallet Fee Discipline (Recommended)

Wallet implementations SHOULD:

* define a configurable premium ceiling (absolute or percentage-based),
* query multiple miners when available,
* select the lowest acceptable premium,
* surface premium cost explicitly to the user.

Wallets MUST preserve the ability for **user-authorized public broadcast** to preserve liveness.

Wallets MAY support explicit user opt-in “private-only” execution modes, but such modes MUST be clearly disclosed and configurable.


### 7.4.5 Failure Handling and Retry Guidance (Informative)

Implementations SHOULD attempt reasonable recovery steps before requiring explicit user authorization or policy intervention.

The following guidance is non-normative and intended to maximize successful private relay while preserving user sovereignty:

**Recommended Sequence:**

1. Retry submission to the same miner in case of transient transport or service errors.
2. Attempt submission to up to *N* alternative miners (configurable; suggested default *N = 3*).
3. Surface the execution state to the user or policy engine with clear options.

**Permitted Outcomes:**

* **RETRY:** Attempt private relay with a different miner (quantum-safe).
* **WAIT:** Defer execution and retry later (quantum-safe).
* **AUTHORIZE PUBLIC BROADCAST:** Proceed with standard broadcast after explicit authorization (quantum-unsafe).
* **ABORT:** Cancel the transaction.

Retries MUST be bounded by wallet-defined timeouts and MUST NOT result in silent deadlock. Implementations MUST clearly disclose when private relay is no longer being attempted and which options remain available.

---

### 7.5 Visibility Guarantees

Seal-360 defines visibility properties during transaction execution that depend
on execution path and authorization state.

| Phase                 | Visibility Scope            |
|----------------------|-----------------------------|
| Pre-execution         | Encrypted payload only      |
| Private relay active  | Selected miner only         |
| Post-confirmation     | Public on-chain             |
| Public broadcast used | Public mempool and network  |

While private relay is active and successful:

* Signed transaction bytes are not broadcast to the public mempool.
* Transaction visibility is restricted to the selected miner.
* Network-level observers do not gain access to transaction signatures.

If public broadcast is explicitly authorized:

* Transaction signatures become publicly observable.
* Execution immediately re-enters the standard Bitcoin threat model.

Seal-360 reduces the observer set during execution when private relay succeeds.
It does not provide anonymity, unlinkability, or protection against miner-side
visibility.

---

### 7.6 Residual Quantum Risk

Seal-360 is designed under the following assumptions:

* Cryptographically relevant quantum computers may become capable of recovering
  ECDSA private keys from exposed signatures.
* ML-KEM-768 and XChaCha20-Poly1305 are not assumed to be breakable within the
  same practical timeframe.
* Symmetric cryptography with ≥256-bit security remains resistant to known
  quantum attacks.

These assumptions align with current NIST post-quantum cryptography guidance.

#### Failure Impact Analysis

If ML-KEM-768 confidentiality is compromised before ECDSA:

* Encrypted Submission confidentiality may be lost.
* Transaction signatures may become observable during private relay.
* No additional execution authority is granted to miners.
* Execution behavior reduces to that of standard Bitcoin broadcast.

If ECDSA is compromised before ML-KEM-768:

* Seal-360 prevents public mempool exposure during the execution window when
  private relay succeeds.
* BIP-360 protects outputs from key-path exposure.
* Combined use preserves execution- and output-layer security under stated
  assumptions.

If both ML-KEM-768 and ECDSA are compromised:

* Seal-360 provides no additional cryptographic protection.
* Execution behavior reduces to standard Bitcoin semantics.

Seal-360 does not claim absolute or permanent quantum immunity.

It provides conditional reduction of execution-window exposure under explicit
assumptions and degrades to standard Bitcoin behavior when those assumptions no
longer hold.


---

### 7.7 Design Rationale and Common Objections (Non-Normative)

This section addresses predictable review objections that arise from misinterpreting Seal-360’s threat model, trust boundaries, and execution semantics.
Nothing in this section introduces new requirements or modifies any normative rule in this specification.

---

#### Does Seal-360 add trust in external parties?

No.

Seal-360 does not introduce any new party that can spend funds, compel execution, or override signer intent.

* The wallet constructs and signs a standard, broadcast-valid Bitcoin transaction.
* The miner receives the transaction only after it is fully signed, and only via an encrypted submission channel.
* The miner does not control signing keys and cannot create an alternative valid spend.
* Any miner refusal, delay, transport failure, or integrity failure triggers wallet execution state surfacing and requires explicit user or policy authorization for any public broadcast.

Seal-360 reduces signature visibility prior to confirmation. It does not transfer authority, custody, or execution control to miners.

---

#### Does Seal-360 centralise transaction execution?

No.

Seal-360 preserves Bitcoin’s censorship resistance and liveness guarantees by ensuring a reachable public broadcast path remains available through explicit authorization.

* Private relay is opportunistic and bounded by wallet-defined timeouts.
* Failure of private relay does not result in automatic public broadcast.
* Users or policy engines explicitly authorize public broadcast when liveness is required.

Seal-360 reduces the observer set during execution. It does not introduce a gatekeeper or single point of control.

---

#### Why is this off-chain instead of on-chain?

Because this is an execution-layer problem that cannot be solved at the consensus layer.

* On-chain mechanisms cannot prevent a signed transaction from being broadcast.
* On-chain mechanisms cannot keep signatures confidential during the execution window.
* Additional consensus or Script changes beyond those introduced by BIP-360 are not required to change delivery semantics between wallets and miners.

Seal-360 operates before public broadcast, where authority timing and signature visibility can be controlled without altering Bitcoin consensus.

---

#### Isn’t this too complex for Bitcoin?

Security mechanisms are evaluated by where complexity is placed.

Seal-360 contains complexity in an optional execution mode at the wallet–miner interface:

* No additional consensus changes beyond BIP-360.
* No additional Script changes beyond BIP-360.
* No transaction validity changes beyond those defined by BIP-360.
* Explicit execution-state handling with user-authorized broadcast.

A minimal implementation requires only discovery, encryption, submission, timeout handling, and state surfacing. More advanced routing or privacy hardening is optional.

This is complexity containment, not complexity expansion.

---

#### What if miners do not adopt Seal-360?

Seal-360 remains correct and live without miner adoption.

* If no eligible miners are discovered, the wallet surfaces execution state and may authorize public broadcast.
* If a miner refuses or does not include within the timeout window, execution state is surfaced for user or policy decision.
* One participating miner already provides value for high-risk transactions. Additional miners reduce correlation and refusal risk.

Ecosystem-wide adoption is not required for correctness or deployability.

---

#### Does Seal-360 weaken Bitcoin’s trust model?

No.

Seal-360 explicitly refuses to claim properties it cannot enforce:

* It does not guarantee inclusion.
* It does not prevent miner censorship.
* It does not provide anonymity.
* It does not provide on-chain post-quantum signatures.

Its sole guarantee is reduced public mempool exposure while private relay succeeds, with user-authorized public broadcast preserving standard Bitcoin semantics.

---

#### Is Seal-360 required for all transactions?

No.

Seal-360 is intended for:

* high-value settlement,
* quantum-sensitive execution,
* institutional or treasury flows,
* environments where execution-window exposure is unacceptable.

Simple transactions remain better served by standard Bitcoin broadcast.

### 7.7.1 Rationale for User-Controlled Execution (Informative)

Seal-360 removes automatic public broadcast to avoid silent degradation of quantum safety during execution.

If a transaction automatically enters the public mempool when private relay fails:

* Transaction signatures become publicly visible.
* A post-quantum adversary gains an execution-window opportunity to extract private keys.
* BIP-360 output hardening becomes ineffective during that window.
* Users seeking quantum safety are downgraded without consent.

Seal-360 therefore separates **mechanism** from **policy**:

* The protocol provides a private execution mechanism.
* Users or operators explicitly choose whether and when to authorize public broadcast.

This design ensures:

* No silent loss of quantum safety.
* Clear accountability for execution decisions.
* Preservation of Bitcoin’s censorship resistance through user-authorized broadcast.
* Strictly better security than standard Bitcoin when quantum-safe execution is selected.

---

## 8. Protocol Overview

Seal-360 replaces immediate public broadcast with **private relay**, while preserving **user-authorized public broadcast** as an execution option.

**Protocol Flow:**
```

1. Discover miner offers
2. Select miner + fee
3. Construct + sign transaction
4. Compute template hash
5. Encrypt transaction (ML-KEM-768)
6. Perform Encrypted Submission to miner
7. Miner decrypts + verifies hash
8. Miner includes transaction in block
9. Transaction becomes public after confirmation

```

Public broadcast is performed only after explicit user or policy authorization.

No transaction is exposed to the public mempool unless public broadcast is authorized.

---

## 9. Protocol Operation

This section defines the complete execution operation for Seal-360, from miner discovery through private relay, inclusion, and user-authorized public broadcast.

---

### 9.1 Miner Discovery

**GOAL:** Locate miners willing to accept private relay submissions.

**INPUTS:**

* None (query phase)

**OUTPUTS:**

* Miner or pool identifier
* Fee terms (sat/vB)
* Public encryption key (ML-KEM-768)
* Offer expiry window
* Optional reputation metadata

**WALLET ACTION:**
Query one or more miners via authenticated endpoints (see Annex H).

**AT THIS STAGE:**

* No inputs selected
* No transaction exists
* No signatures produced
* No broadcast-valid artefact exists

An execution offer communicates the terms under which a miner is willing to accept Encrypted Submission.

---

#### 9.1.1 Provider Metadata and Endpoint Validation

Miners MAY publish optional metadata to assist wallets in routing decisions, including:

- verifiable miner or pool identifiers,
- historical inclusion reliability,
- jurisdictional disclosure,
- data-handling or privacy commitments,
- signed endpoint attestations.

Wallets SHOULD perform basic validation to reduce Sybil or impersonation risk, including:

- de-duplication of miner identities
- verification of long-lived encryption keys
- consistency checks between advertised identity and observed behaviour
- correlation of miner identity with on-chain hashpower evidence (where available)

Wallets MAY query multiple miners in parallel to reduce correlation and refusal risk.

---

#### 9.1.2 Public Key Authenticity

Wallets MUST verify the authenticity of the miner’s public encryption key.

Public keys MUST be obtained via an authenticated channel, such as:

* HTTPS with a valid TLS certificate
* DNSSEC-secured DNS records
* a signed provider manifest (Annex H)

If authenticity cannot be established, submission MUST be rejected.

---

### 9.2 Transaction Construction

This section defines the construction of a broadcast-valid Bitcoin transaction
prior to encrypted private relay.

Transaction construction occurs entirely within the wallet and precedes any
Encrypted Direct-to-Miner Submission.

---

#### Goal

Construct a fully signed, broadcast-valid Bitcoin transaction without exposing
the transaction to the public mempool.

---

#### Inputs

- Selected miner offer (Section 9.1)
- Wallet-controlled UTXOs
- Recipient output specifications
- Wallet fee and change policy

---

#### Outputs

- Fully signed Bitcoin transaction suitable for public broadcast

---

#### Wallet Requirements

The wallet MUST:

1. Select inputs according to wallet policy.
2. Define outputs, including destination and change outputs.
3. Calculate transaction fees based on the selected miner offer.
4. Construct the transaction using standard Bitcoin serialization rules.
5. Sign the transaction using standard Bitcoin signing rules.

The resulting transaction MUST be valid under Bitcoin consensus rules and MUST
be broadcast-valid if public broadcast is later authorized.

---

#### Execution Ordering Constraint

After signing and before completion of Encrypted Submission, the wallet
MUST NOT broadcast the transaction to the public peer-to-peer network or
public mempool.

This prohibition applies regardless of:

- timeout configuration,
- miner availability,
- transport failure,
- application restart or recovery,
- policy engine state.

Any implementation that broadcasts a transaction at this stage is
non-conformant.

---

#### Mutation Prohibition

After signing:

- the transaction bytes MUST be treated as immutable,
- no inputs, outputs, scripts, witnesses, fees, or ordering MAY be modified,
- any change requires explicit reconstruction and re-signing.

Any modification after signing invalidates the transaction instance and MUST
result in a new transaction with a new `template_hash`.

---

#### Scope Clarification

This section defines transaction construction only.

It does not define:

- authorization semantics,
- execution policy,
- miner inclusion behavior,
- public broadcast conditions.

Those behaviors are defined elsewhere in this specification.

---

## 9.3 Template Hashing (Normative)

Template hashing binds an Encrypted Submission to a single, immutable Bitcoin
transaction instance.

The template hash prevents undetected modification of transaction bytes after
signing and ensures that miners gain visibility but not control.

---

### 9.3.1 Canonical Hash Input (Normative)

`template_hash` MUST be computed over the exact transaction byte sequence that
would be broadcast if public broadcast is later authorized.

The hash input MUST:

- use standard Bitcoin wire serialization, and
- include all witness data exactly as signed.

No metadata, wrappers, JSON, CBOR, transport framing, or auxiliary fields MAY be
included.

---

### 9.3.2 Serialization Requirements (Normative)

The wallet MUST hash the transaction bytes exactly as signed, including:

- `version` (4 bytes, little-endian),
- all inputs (outpoint, scriptSig, sequence),
- all outputs (value, scriptPubKey),
- witness stacks (if present, exactly as signed),
- `locktime` (4 bytes, little-endian).

The following transformations are FORBIDDEN:

- witness stripping or reordering,
- signature normalization or rewriting,
- input or output reordering,
- fee or script mutation,
- any byte-level rewrite.

Any post-signing modification MUST produce a different `template_hash`.

---

### 9.3.3 Hash Function (Normative)

```

template_hash = SHA256(raw_tx_bytes)

```

The output MUST be exactly 256 bits (32 bytes).

---

### 9.3.4 Binding Properties (Normative)

The template hash binds, at minimum:

- transaction version,
- all inputs and outputs,
- all signatures and witness data,
- locktime.

Any modification to any bound field MUST invalidate the hash.

---

### 9.3.5 Miner Verification Rule (Normative)

After decryption, the miner MUST:

1. Recompute `template_hash` over the decrypted transaction bytes.
2. Compare it to the supplied `template_hash`.
3. Reject the submission if the values differ.

A mismatch MUST be treated as a hard integrity failure.

---

### 9.3.6 Spend Form Constraint (Normative)

Seal-360 MUST be used only with transaction forms that are not subject to
third-party malleability.

Conformant wallets MUST restrict Seal-360 execution to:

- Taproot (SegWit v1) spends, or
- any future Bitcoin spend form with equivalent non-malleability guarantees.

Seal-360 MUST NOT be used with legacy or SegWit v0 inputs whose witness data may
be altered by third parties without invalidating the spend.

This constraint is REQUIRED to ensure that `template_hash` uniquely commits to a
single valid transaction instance.

---

### 9.4 Miner Offer Object (Informational)

Miners MAY publish execution offers prior to receiving transactions.

Offers are informational only and MUST NOT be treated as inclusion guarantees.

The `template_hash` is NOT included in offers. It is supplied only after signing and encryption.

---

### 9.5 Transaction Encryption

**GOAL:** Encrypt the transaction for quantum-safe private relay.

**INPUTS:**

* Signed transaction bytes
* Miner public key (ML-KEM-768)
* `template_hash`

**OUTPUTS:**

* Encrypted Submission package

---

### 9.5.1 Recommended Encryption Profile (Normative)

Seal-360 Encrypted Submission MUST use the following cryptographic profile:

* ML-KEM-768 (FIPS 203) for key encapsulation
* XChaCha20-Poly1305 for AEAD encryption
* Ephemeral encapsulation per submission attempt

The wallet MUST generate a fresh `submission_id` for each Encrypted Submission
attempt.

`submission_id` MUST be exactly 16 bytes of cryptographically secure random
data and MUST NOT be reused across submissions, miners, or retry attempts.

The wallet MUST include AEAD associated data (AAD) constructed exactly as
specified in Section 9.5.3.

Any deviation from the canonical AAD construction or encoding rules defined in
Section 9.5.3 MUST cause the miner to reject the submission.

The encryption profile defined in this section applies only to the Encrypted
Submission payload and does not alter Bitcoin transaction construction,
signing, or validation semantics.

---

#### 9.5.2 Transitional Encryption Profile

Classical ECIES-secp256k1 MAY be supported for compatibility but is NOT quantum-resistant and SHOULD be avoided where post-quantum protection is required.

---

#### 9.5.3 Canonical AAD Encoding (Normative)

AEAD associated data (AAD) MUST be constructed as a canonical encoding over the
following fields and no others:

* `template_hash` — 32-byte SHA-256 hash of the signed transaction bytes,
  hex-encoded
* `submission_id` — 16-byte per-attempt identifier, hex-encoded
* `miner_id` — UTF-8 string identifying the selected miner or pool
* `key_id` — UTF-8 string identifying the miner encryption key
* `expires_at` — RFC 3339 UTC timestamp defining the validity window of the offer

AAD MUST be encoded using RFC 8785 JSON Canonicalization Scheme (JCS).

The canonical JSON object MUST contain exactly the fields listed above and MUST
NOT include any additional fields, metadata, or transport-specific information.

Example (illustrative only):

```json
{"expires_at":"2026-02-15T00:00:00Z","key_id":"2026-01-key-01","miner_id":"example-pool-01","submission_id":"00112233445566778899aabbccddeeff","template_hash":"0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef"}
```

Miners MUST reconstruct the canonical AAD using the same field set and encoding
rules and MUST verify AEAD integrity under that AAD.

Any AEAD validation failure, field mismatch, replayed `submission_id`, expired
`expires_at`, or non-canonical encoding MUST result in rejection of the
submission.

---

### 9.6 Encrypt-Before-Transport Semantics (Normative)

Seal-360 mandates **Encrypt-Before-Transport (EBT)** semantics.

The signed transaction payload MUST be cryptographically encrypted **before**
entering any transport, framing, routing, or delivery mechanism.

All transports are treated as metadata-exposing and untrusted for confidentiality.

Transport mechanisms (including HTTPS, TLS, KeyMail, binary framing, or any
other carrier) MUST carry only ciphertext and MUST NOT be relied upon for
payload secrecy.

EBT defines payload confidentiality only.  
EBT does not define framing, routing, retry, ordering, or delivery guarantees.

Transport choice MUST NOT affect:
- transaction construction or signing,
- `template_hash` computation,
- AEAD associated data construction,
- authorization semantics,
- fallback or failure behavior.

Seal-360 defines payload structure, cryptographic binding, and execution
semantics independently of transport.

#### 9.6.1 HTTPS/TLS as Authenticated Carrier (RECOMMENDED)

HTTPS over TLS 1.3 or later is RECOMMENDED as an authenticated transport
carrier for Encrypted Submission.

When HTTPS/TLS is used:

- the miner submission endpoint MUST be authenticated using standard TLS
  certificate validation,
- TLS is used solely to authenticate the endpoint and protect against trivial
  network interference,
- post-quantum confidentiality is provided exclusively by Encrypt-Before-Transport
  payload encryption.

Seal-360 MUST NOT rely on TLS for payload confidentiality.

Compromise or failure of TLS authentication MAY enable endpoint impersonation
or denial-of-service, but MUST NOT:

- expose signed transaction bytes,
- alter payload integrity without detection,
- trigger automatic public broadcast,
- grant execution or signing authority.

#### 9.6.2 Optional Binary Framing (Framing Only)

Implementations MAY use binary framing instead of JSON- or REST-based submission
to carry Encrypted Submission payloads.

When binary framing is used:

- the encrypted payload defined in Section 9.5 MUST remain unchanged,
- AEAD associated data construction per Section 9.5.3 MUST remain unchanged,
- framing MUST be length-delimited and unambiguous,
- no additional plaintext transaction metadata MAY be introduced.

Binary framing affects framing only.

Binary framing MUST NOT:
- alter Encrypt-Before-Transport semantics,
- introduce new execution behavior,
- introduce new authorization semantics,
- weaken payload confidentiality or integrity guarantees.

#### 9.6.3 Carrier Selection and Failure Handling

Wallets MAY support multiple transport mechanisms concurrently.

Recommended behavior:

* attempt Encrypted Submission using the preferred configured transport,
* on transport failure, retry using an alternative authenticated transport where available,
* surface execution state if all configured transports fail.

Transport failure MUST NOT:

* modify payload contents,
* alter AEAD associated data,
* trigger public broadcast automatically,
* bypass Explicit Public Broadcast Authorization Event requirements.

#### 9.6.4 Carrier Security Scope

Seal-360 provides post-quantum confidentiality through
Encrypt-Before-Transport (EBT) payload encryption.

Seal-360 does **not** claim post-quantum security for:

- carrier authentication mechanisms,
- endpoint discovery infrastructure,
- classical PKI systems (for example TLS certificate authorities),
- network-layer metadata such as IP addresses, routing information,
  timing, or packet size.

Authenticated carriers such as HTTPS/TLS are used solely to identify
submission endpoints and to protect against trivial network
interference. They are **not** relied upon for payload confidentiality.

Compromise or failure of carrier authentication MAY enable endpoint
impersonation, traffic disruption, or denial-of-service. Such failures:

- MUST NOT grant execution authority or signing capability,
- MUST NOT permit modification of signed transaction bytes without
  detection,
- MUST NOT trigger automatic public broadcast.

Private relay failure resulting from carrier issues is handled as an
execution failure and requires explicit user or policy decision before
any public broadcast.

---

### 9.7 Miner Processing (Normative)

Upon receiving an Encrypted Submission, a miner claiming Seal-360 conformance
MUST perform the following steps in order:

1. Validate submission structure and required fields.
2. Enforce replay protection by rejecting any submission with a previously
   observed (`miner_id`, `submission_id`) tuple that has not yet expired.
3. Verify that `expires_at` has not passed.
4. Decapsulate the shared secret using the miner’s ML-KEM-768 private key.
5. Derive the AEAD encryption key from the shared secret.
6. Reconstruct the canonical AEAD associated data (AAD) exactly as specified
   in Section 9.5.3.
7. Perform AEAD decryption using the reconstructed AAD.
8. Recompute `template_hash` over the decrypted transaction bytes.
9. Reject the submission if the recomputed `template_hash` does not match the
   supplied value.
10. Validate the transaction under standard Bitcoin consensus rules.
11. If accepted, include the transaction in a candidate block according to
    the miner’s normal fee and policy logic.

Replay protection MUST be enforced for the duration of the offer validity window
and MUST survive miner restarts where feasible.

The miner MUST NOT modify transaction bytes, create alternative spends, or
assume any execution or broadcast authority.

Miner refusal, delay, or non-inclusion is permitted and does not constitute a
protocol violation.

---

### 9.8 Public Broadcast Authorization

Public broadcast is an explicit execution choice and MUST NOT be triggered
automatically by the Seal-360 protocol.

A transaction MAY be submitted to the public Bitcoin peer-to-peer network only
after an Explicit Public Broadcast Authorization Event as defined in
Section 11.4.

Public broadcast MAY be requested when one or more of the following occur:

- the selected miner refuses the Encrypted Submission,
- inclusion does not occur within the wallet-defined timeout,
- the submission endpoint becomes unreachable,
- an integrity, authentication, or protocol failure is detected during private
  relay.

On any such condition, the wallet MUST surface execution state to the user or
policy engine and MUST offer the execution outcomes defined in Section 11.1.

#### Prohibited Behavior

The following conditions MUST NOT, by themselves, cause public broadcast:

- timeout expiration,
- miner refusal,
- transport failure,
- discovery failure,
- integrity failure,
- application restart or recovery.

Any policy or automation that results in public broadcast MUST satisfy the
requirements of Section 11.4 and MUST be auditable per Section 11.5.

When public broadcast is authorized, the wallet MUST broadcast the exact signed
transaction bytes that were hashed to produce `template_hash`, unless a new
transaction is explicitly constructed as part of downgrade handling.

---

### 9.8.1 Correlation Risk

Transitioning from private relay to public broadcast introduces correlation
risk.

Potential correlation vectors include, but are not limited to:

- timing correlation between private relay attempts and public broadcast,
- fee structure changes or premiums,
- miner selection history,
- transaction size, shape, and input/output structure.

Seal-360 does not claim anonymity, unlinkability, or metadata privacy.

Correlation risk is inherent when execution transitions from a private delivery
path to the public network and cannot be fully eliminated.

---

### 9.8.2 Correlation Downgrade Guard

When transitioning from private relay to authorized public broadcast, wallets
MUST apply a downgrade guard to reduce deterministic linkage between the two
execution paths.

At minimum, a conformant wallet MUST perform at least one of the following
actions prior to public broadcast:

- introduce a randomized delay of at least one block interval,
- reconstruct the transaction with a new fee and fresh signatures,
- verify that the transaction is not already present in the public mempool.

Wallets MUST NOT immediately rebroadcast a privately relayed transaction without
applying a downgrade guard.

The specific downgrade guard strategy is implementation-defined, but MUST be
documented and MUST be applied consistently.

Failure to apply a downgrade guard constitutes non-conformance.

---

### 9.9 Post-Execution Output Anchoring

Wallets SHOULD prefer BIP-360 P2TSH outputs for both destination and change to maintain output-layer hardening.

---

### 9.10 Replace-By-Fee (RBF)

Standard Bitcoin RBF semantics apply.

Higher-fee replacements MAY be submitted via private relay or user-authorized public broadcast.

---

### 9.10.1 RBF Semantics Under Private Relay (Normative)

Seal-360 does not modify Bitcoin’s Replace-By-Fee (RBF) rules.

During private relay:

- A miner MAY receive a privately relayed transaction that signals RBF.
- A miner MAY later observe a higher-fee replacement transaction for the same
  inputs via public broadcast or another private relay path.
- Miners remain free to apply standard Bitcoin RBF policy when selecting
  transactions for inclusion.

Seal-360 provides no guarantee that a privately relayed transaction will be
preferred over a later higher-fee replacement.

Wallet implications:

- Wallets SHOULD assume that privately relayed transactions are not immune to
  later replacement under standard RBF rules.
- Wallets SHOULD treat private relay as an execution visibility mechanism, not
  a replacement-prevention mechanism.
- If strong replacement resistance is required, wallets MUST rely on standard
  Bitcoin mechanisms outside the scope of this specification.

Private relay does not create a new replacement class and does not constrain
miner fee or ordering policy.

---

### 9.11 Normative Scope

All guarantees in this section apply **only while private relay is active**.

Once public broadcast is explicitly authorized, standard Bitcoin threat assumptions apply immediately.

Seal-360 introduces no guarantees beyond those explicitly stated and does not modify Bitcoin consensus, relay, or execution semantics beyond those defined by BIP-360.

---

## 10. Security Properties

### 10.1 Guaranteed

When used as specified, Seal-360 provides the following guarantees during private
relay execution:

* **No public mempool exposure during successful private relay**  
  While private relay is active and successful, the signed transaction MUST NOT
  be broadcast to the public mempool and MUST NOT be observable by public network
  participants.

* **Reduction of observer set during execution**  
  During private relay, transaction visibility is reduced from the global Bitcoin
  network to the selected miner. This is a reduction in observer set size, not a
  claim of anonymity.

* **Post-quantum payload confidentiality during submission**  
  Encrypted Submission protects transaction payload confidentiality during
  delivery under the stated cryptographic assumptions.

* **Transaction integrity enforcement**  
  Any modification of the transaction bytes after signing is detectable via
  template hash verification and MUST result in rejection. This prevents
  undetected byte-level tampering, but does not prevent refusal, delay, leakage,
  or economic coercion.

* **Explicit user-controlled liveness**  
  A valid transaction always retains a reachable public broadcast path, but
  public broadcast is permitted only after an Explicit Public Broadcast
  Authorization Event (Section 11.4) and is never triggered automatically by
  timeout, failure, or recovery.

* **No silent downgrade of execution security**  
  Failure of private relay results in surfaced execution state and explicit user
  or policy decision-making. Quantum-safe execution is never silently converted
  into public broadcast.

* **Crash and restart safety**  
  Application crashes, restarts, or recovery procedures MUST NOT cause public
  broadcast. Pending transactions remain non-broadcast until explicitly
  authorized.

* **Consensus and relay compatibility**  
  Seal-360 introduces no additional changes to Bitcoin consensus rules, Script semantics, mempool policy, or relay behavior beyond those required by BIP-360.

All guarantees in this section apply only while private relay is active and
successful. Once public broadcast is explicitly authorized, execution
immediately re-enters the standard Bitcoin threat model.

### 10.2 Not Guaranteed

Seal-360 explicitly does **not** guarantee the following properties:

* **Miner inclusion or timeliness**  
  Miners may refuse, delay, reorder, or ignore any Encrypted Submission for any
  reason. Seal-360 provides no inclusion, ordering, or confirmation guarantees.

* **Censorship resistance during private relay**  
  A selected miner may censor or withhold a transaction during private relay.
  Bitcoin’s censorship resistance is preserved only through user-authorized
  public broadcast.

* **Anonymity or metadata privacy**  
  Seal-360 reduces the observer set during execution but does not provide
  anonymity, unlinkability, or protection against timing, fee, or metadata
  correlation. Miner-side visibility may allow inference.

* **Protection against miner leaks**  
  A miner may deliberately or accidentally leak a decrypted transaction prior to
  confirmation. Such a leak constitutes a privacy and execution-timing failure,
  not a protocol or execution error, and does not grant the miner execution
  authority.

* **Protection after public broadcast**  
  Once public broadcast is explicitly authorized and performed, transaction
  signatures become publicly observable and standard Bitcoin execution risks
  apply immediately.

* **On-chain post-quantum signatures**  
  Seal-360 does not modify Bitcoin’s signature scheme and does not provide
  post-quantum signatures on-chain. Output-layer quantum hardening requires
  BIP-360 (P2TSH).

* **Protection against miner collusion**  
  Colluding miners sharing private relay information are out of scope and may
  defeat observer-set reduction.

* **Fee market outcomes**  
  Seal-360 does not guarantee confirmation speed, fee efficiency, fee fairness,
  or resistance to fee premium extraction beyond standard Bitcoin fee-market
  dynamics.

* **Network-layer privacy**  
  Transport-layer encryption protects submission payloads but does not conceal
  network metadata such as IP addresses, routing characteristics, or endpoint
  access patterns.

Failed private relay followed by user-authorized public broadcast may introduce
additional delay or correlation relative to immediate public broadcast.

Seal-360 is an execution-layer mitigation only. It reduces a specific
execution-window attack surface and does not redefine Bitcoin’s trust model,
threat model, or consensus assumptions.

### 10.3 Security Properties Summary

| Property                               | Provided | Notes |
|----------------------------------------|----------|------|
| No public mempool exposure             | Yes      | While private relay succeeds |
| Quantum-resistant payload confidentiality | Yes   | Encrypt-Before-Transport (EBT) using ML-KEM-768 + AEAD |
| Transaction integrity                  | Yes      | Template hash binding |
| User-controlled liveness               | Yes      | Explicit public broadcast authorization |
| Miner inclusion guarantee              | No       | Miner may refuse or delay |
| Anonymity                              | No       | Observer set reduced, metadata not concealed |
| Protection after public broadcast      | No       | Standard Bitcoin exposure applies |
| On-chain quantum safety                | No       | Requires BIP-360 (P2TSH) |

Seal-360 provides quantum-resistant confidentiality for transaction payloads
**prior to confirmation** through Encrypt-Before-Transport semantics.  
It does not rely on transport mechanisms for confidentiality and does not alter
Bitcoin’s on-chain cryptography.

All guarantees above apply only while private relay is active and successful.
Once public broadcast is explicitly authorized, execution immediately re-enters
the standard Bitcoin threat model.

### 10.4 Security Comparison (Informative)

This table compares execution behavior under standard Bitcoin broadcast,
BIP-360 alone, and Seal-360 under different execution outcomes.

| Execution Mode                          | Public Mempool Exposure        | Execution-Window Protection | Payload Confidentiality | Liveness Characteristics |
|----------------------------------------|--------------------------------|-----------------------------|-------------------------|--------------------------|
| Standard Bitcoin                        | Always                         | None                        | None                    | Immediate                |
| BIP-360 Only (P2TSH)                   | Always                         | None                        | None                    | Immediate                |
| Seal-360 (private relay succeeds)      | None prior to confirmation     | Yes                         | Yes (EBT)               | Conditional              |
| Seal-360 (public broadcast authorized) | After explicit authorization  | None                        | None                    | Equivalent to standard   |

Interpretation:

- Standard Bitcoin and BIP-360 alone expose signatures to the public mempool
  during the execution window.
- Seal-360 removes public mempool exposure during execution only while private
  relay succeeds.
- Encrypt-Before-Transport (EBT) provides quantum-resistant **payload
  confidentiality**, not transport-level security.
- When public broadcast is explicitly authorized, Seal-360 execution behavior
  becomes equivalent to standard Bitcoin for that transaction.

Seal-360 alters **delivery timing and visibility**, not transaction validity,
consensus rules, or broadcast availability.

---

## 11. Failure Modes

Seal-360 defines explicit, bounded failure modes. All failure modes preserve user sovereignty and MUST NOT result in silent loss, automatic broadcast, or indefinite limbo.

| Condition              | Detection Point                | Required Action           | Result                                     |
| ---------------------- | ------------------------------ | ------------------------- | ------------------------------------------ |
| Miner refusal          | Encrypted Submission response  | Surface decision          | Retry, wait, authorize broadcast, or abort |
| Inclusion timeout      | Local timeout exceeded         | Surface decision          | Retry, wait, authorize broadcast, or abort |
| Miner unreachable      | Transport failure              | Retry or surface decision | Retry, wait, authorize broadcast, or abort |
| Template hash mismatch | Miner verification             | Reject submission         | Retry or surface decision                  |
| Encryption failure     | Wallet-side error              | Abort private relay       | Surface decision                           |
| Discovery failure      | No eligible miners             | Skip private relay        | Surface decision                           |
| Policy deadlock        | Policy engine blocks execution | Surface refusal           | User or policy decision required           |

---

### 11.1 User Sovereignty Over Execution (Normative)

For any broadcast-valid transaction:

* The user MUST retain the ability to manually authorize public broadcast to the Bitcoin network.
* No failure mode may permanently prevent user-initiated public broadcast.
* No external system may silently absorb, discard, or auto-broadcast a transaction without an Explicit Public Broadcast Authorization Event as defined in Section 11.4.
* Wallets MUST clearly present available execution options when private relay fails:

  * **RETRY:** Attempt private relay with a different miner (quantum-safe).
  * **WAIT:** Defer execution and retry later (quantum-safe).
  * **AUTHORIZE PUBLIC BROADCAST:** Proceed with standard broadcast after explicit authorization (quantum-unsafe).
  * **ABORT:** Cancel the transaction.

Any implementation that causes public broadcast without an Explicit Public Broadcast Authorization Event, or that prevents user-initiated broadcast, is non-conformant.

---

### 11.2 Failure Transparency

Wallets MUST surface failure conditions clearly to users or policy engines, including:

* the reason for private relay failure,
* whether quantum-safe execution remains possible,
* the implications of authorizing public broadcast.

Failure handling MUST favor explicit user or policy choice over automation.

### 11.3 Safety Property

Seal-360 failure modes preserve explicit user control over the tradeoff between
quantum exposure and transaction liveness.

When quantum-safe execution options are selected:

* Failure results in non-confirmation rather than automatic public exposure.
* No signed transaction is broadcast to the public mempool without explicit
  authorization.

When public broadcast is explicitly authorized:

* The transaction follows standard Bitcoin broadcast and confirmation behavior.
* Execution semantics, exposure, and risk align with those of ordinary public
  broadcast.

Seal-360 does not introduce new execution failure modes beyond delayed
confirmation, user cancellation, or explicitly authorized public broadcast.

All transitions between private relay and public broadcast are explicit,
auditable, and initiated by the user or a pre-armed policy.

### 11.4 Explicit Public Broadcast Authorization Event (Normative)

An Explicit Public Broadcast Authorization Event MUST be exactly one of the following:

**(A) Interactive User Confirmation**
* A user confirmation step performed after signing that explicitly authorizes public broadcast for this specific transaction.
* The confirmation MUST be attributable to the user (local UI confirmation or equivalent secure approval mechanism).
* A generic “OK” or background approval that does not clearly indicate public broadcast authorization MUST NOT qualify.

**(B) Pre-Armed Policy Authorization**
* A pre-registered, explicit policy rule that authorizes public broadcast under specified conditions.
* The policy MUST be configured before the wallet initiates private relay for the transaction.
* The policy MUST NOT default to allow. Absence of a policy MUST be treated as “no authorization.”
* A policy MAY authorize broadcast after timeout only if the policy explicitly includes a broadcast-after-timeout rule.

**Forbidden interpretations:**
* “Timeout occurred” is not authorization.
* “Policy engine decided” is not authorization unless it satisfies (B).
* “App restart recovery” is not authorization.

### 11.5 Authorization Audit Record (Normative)

If public broadcast occurs under (A) or (B), the wallet MUST record a local Authorization Audit Record including:

* transaction identifier (txid if known, otherwise `template_hash`)
* authorization type: `USER_CONFIRMATION` or `PRE_ARMED_POLICY`
* timestamp (wall-clock is acceptable)
* if policy-based: policy identifier and the condition that evaluated to true

The Authorization Audit Record MUST remain local unless explicitly exported by the user.

### 11.6 Network Partition Handling

Seal-360 implementations MUST account for network partitions, delayed block
propagation, and partial visibility of confirmation state.

A network partition MAY cause a wallet to temporarily lose visibility into
blocks mined by a selected miner while the miner continues normal operation.

---

#### Partition Safety Requirement

Before authorizing public broadcast following a private relay attempt, the
wallet MUST perform at least one of the following checks:

- verify that the transaction is not already present in the public mempool,
- verify that the transaction has not already confirmed on-chain,
- verify that the selected miner has not advertised inclusion of the
  transaction in a candidate block.

Failure to perform a partition safety check constitutes non-conformance.

---

#### Double-Submission Prevention

A wallet MUST NOT authorize public broadcast if it detects that:

- the transaction has already confirmed, or
- the transaction is already visible in the public mempool.

If confirmation state is ambiguous, the wallet MUST surface execution state and
require explicit user or policy decision before proceeding.

---

#### Idempotency Constraint

The wallet MUST treat a signed transaction instance as idempotent across
private relay and public broadcast paths.

The same signed transaction instance MUST NOT be submitted via multiple
execution paths unless an Explicit Public Broadcast Authorization Event
(Section 11.4) has occurred.

If a new transaction is constructed for public broadcast as part of downgrade
handling, it MUST be treated as a distinct transaction instance and MUST be
re-signed, producing a new `template_hash`.

---

## 12. Deployment Requirements

### 12.1 Wallet Requirements

A wallet claiming Seal-360 conformance MUST:

* construct standard, broadcast-valid Bitcoin transactions using existing consensus rules,
* compute `template_hash` exactly as specified in Section 9.3,
* encrypt transactions for Encrypted Direct-to-Miner Submission using the required cryptographic profile,
* verify miner public encryption keys and manifest authenticity prior to submission,
* perform Encrypted Submission only after a transaction is fully signed,
* enforce bounded retry and timeout behavior during private relay,
* surface execution state and failure reasons to the user or policy engine on any private relay failure,
* require an Explicit Public Broadcast Authorization Event (Section 11.4) before any public broadcast,
* record an Authorization Audit Record (Section 11.5) for any authorized public broadcast,
* ensure crash or restart recovery restores pending transactions to a non-broadcast state until an Explicit Public Broadcast Authorization Event occurs,
* preserve user-initiated public broadcast as a reachable execution option at all times.

A wallet claiming Seal-360 conformance MUST NOT:

* automatically broadcast a transaction to the public mempool due to timeout, refusal, transport failure, discovery failure, restart recovery, or integrity failure,
* treat the absence of a policy, a default policy, or an implicit timeout as authorization to broadcast,
* modify signed transaction bytes after `template_hash` computation,
* suppress, bypass, or delay user-authorized public broadcast,
* delegate execution authority, signing control, or broadcast authority to miners,
* introduce execution deadlock that prevents either private relay retry or user-authorized public broadcast.

Wallets MAY implement policy-based or automated decision engines, but any policy-triggered public broadcast MUST:

* satisfy the requirements of Section 11.4(B),
* be configured prior to initiating private relay for the transaction, and
* produce an Authorization Audit Record in accordance with Section 11.5.

Any wallet that causes public broadcast without an Explicit Public Broadcast Authorization Event, prevents user-initiated broadcast, auto-broadcasts on failure, or alters signed transaction bytes post-hash is non-conformant.

---

### 12.2 Miner Requirements

A miner claiming Seal-360 conformance MUST satisfy all requirements in this
section.

Failure to meet any requirement in this section constitutes non-conformance.

---

#### 12.2.1 Key Publication and Authenticity

The miner MUST publish a valid post-quantum public encryption key suitable for
Encrypted Direct-to-Miner Submission.

- The key MUST be ML-KEM-768 as defined in FIPS 203.
- The key MUST be discoverable through an authenticated discovery mechanism
  defined in Annex H.
- Each key MUST be identified by a stable `key_id`.

The miner MUST NOT accept Encrypted Submissions encrypted to expired or
revoked keys.

---

#### 12.2.2 Encrypted Submission Acceptance

The miner MUST accept Encrypted Submissions only over authenticated transport
mechanisms.

Transport authentication is used solely to identify the submission endpoint.
The miner MUST NOT rely on transport-layer confidentiality for payload secrecy.

All received payloads MUST be treated as untrusted until cryptographic
verification succeeds.

---

#### 12.2.3 Replay Protection (Mandatory Persistence)

The miner MUST implement replay protection for Encrypted Submissions.

Replay protection MUST be enforced using the tuple:

- (`miner_id`, `submission_id`)

Replay state MUST satisfy all of the following:

- Replay state MUST be recorded upon first successful decryption attempt.
- Replay state MUST be persisted across miner restarts.
- Replay state MUST be retained for at least the full duration of the offer
  validity window defined by `expires_at`.
- In-memory-only replay caches are NOT permitted.

Any Encrypted Submission with a previously observed
(`miner_id`, `submission_id`) tuple that has not yet expired MUST be rejected.

---

#### 12.2.4 Offer Expiry Enforcement

The miner MUST verify that `expires_at` has not passed before attempting
decryption.

Expired offers MUST be rejected prior to cryptographic processing.

---

#### 12.2.5 Cryptographic Processing

Upon receiving an Encrypted Submission, the miner MUST perform the following
steps in order:

1. Validate submission structure and required fields.
2. Enforce replay protection.
3. Verify that `expires_at` has not passed.
4. Decapsulate the shared secret using the miner’s ML-KEM-768 private key.
5. Derive the AEAD encryption key from the shared secret.
6. Reconstruct the canonical AEAD associated data (AAD) exactly as specified
   in Section 9.5.3.
7. Perform AEAD decryption using the reconstructed AAD.
8. Recompute `template_hash` over the decrypted transaction bytes.
9. Reject the submission if the recomputed `template_hash` does not match the
   supplied value.

Any failure at any step MUST result in rejection of the submission.

---

#### 12.2.6 Transaction Validation and Inclusion

If cryptographic verification succeeds, the miner MUST validate the transaction
under standard Bitcoin consensus rules.

If the transaction is valid, the miner MAY include it in a candidate block
according to the miner’s normal fee, policy, and ordering logic.

Seal-360 does NOT impose any obligation to include a transaction.

---

#### 12.2.7 Non-Authority Constraint

The miner MUST NOT:

- modify transaction bytes after decryption,
- create or attempt to create alternative valid spends,
- assume custody, signing, execution, or broadcast authority,
- trigger or initiate public broadcast.

Miner refusal, delay, censorship, or non-inclusion is permitted and does not
constitute a protocol violation.

---

#### 12.2.8 Failure Semantics

Any rejection, refusal, delay, or non-inclusion by the miner:

- MUST be treated as an execution failure,
- MUST NOT trigger automatic public broadcast,
- MUST NOT modify transaction bytes,
- MUST NOT assert execution authority.

All public broadcast decisions remain exclusively with the wallet or policy
engine and require an Explicit Public Broadcast Authorization Event
(Section 11.4).

---

#### 12.2.9 Conformance Boundary

A miner MAY advertise Seal-360 support only if all requirements in this section
are met.

Partial, best-effort, or transport-only implementations MUST NOT claim
Seal-360 conformance.

---

### 12.3 Bitcoin Node Requirements

Bitcoin nodes require no changes beyond those required to support BIP-360.

Seal-360 operates exclusively at the wallet–miner interface and does not modify:

- block or transaction validation rules,
- Script semantics,
- mempool policy,
- peer-to-peer relay behavior.

Transactions delivered via Seal-360 private relay are validated identically to
any other BIP-360-compliant transactions once included in a block.

Bitcoin nodes are not required to distinguish Seal-360-delivered transactions
from standard transactions.

---

### 12.4 Optional Infrastructure

Deployments MAY include:

* miner reputation tracking,
* multi-endpoint discovery,
* encrypted relay gateways,
* offline signing coordination.

Such infrastructure MUST NOT prevent user-authorized public broadcast or introduce execution deadlock.

Optional infrastructure MUST NOT:

* automatically trigger public broadcast,
* alter signed transaction bytes,
* assert miner inclusion guarantees,
* introduce new trust assumptions,
* modify Bitcoin consensus or relay behavior.

---

### 12.5 Backward Compatibility

Seal-360 is backward compatible with all existing Bitcoin wallets and nodes.

Transactions produced under Seal-360 are indistinguishable from standard Bitcoin transactions once confirmed on-chain.

---

## 13. Summary

BIP-360 hardens outputs.  
Seal-360 addresses execution.

Seal-360 is an execution-layer overlay that reduces public mempool exposure
during the period between transaction signing and confirmation by enabling
encrypted, direct-to-miner submission of fully signed Bitcoin transactions.

Seal-360 does not modify:

* Bitcoin consensus rules,
* Script semantics,
* transaction validity,
* custody or signing authority,
* miner obligations or guarantees.

While private relay succeeds, transaction signatures are not publicly observable
prior to confirmation. During this period, execution-window exposure is reduced
to the selected miner.

If private relay does not succeed, execution state is surfaced and public
broadcast remains available only through explicit user or policy authorization.
Once authorized, execution proceeds under standard Bitcoin broadcast semantics.

Seal-360 preserves Bitcoin’s censorship resistance by ensuring that public
broadcast remains reachable at all times through explicit authorization, while
preventing automatic or silent exposure during execution.

Seal-360 is an opt-in execution-layer mitigation that composes cleanly with and assumes the presence of BIP-360, providing execution-window protection on top of BIP-360’s output hardening.

---

## 14. Conformance Statement

An implementation MAY claim **Seal-360 conformance** only if all mandatory requirements in this specification are met.

### 14.1 Wallet Conformance

A wallet implementation claiming Seal-360 conformance MUST:

* construct broadcast-valid Bitcoin transactions,
* compute `template_hash` per Section 9.3,
* encrypt transactions for Encrypted Direct-to-Miner Submission using the required cryptographic profile,
* verify miner public keys before submission,
* enforce bounded retry and timeout behavior during private relay,
* surface execution state and failure reasons to the user or policy engine,
* require explicit user or policy authorization before any public broadcast,
* prevent silent drop or indefinite execution deadlock.

Wallets that prevent user-authorized public broadcast, alter signed transactions post-hash, or allow indefinite execution deadlock are non-conformant.

---

### 14.2 Miner Conformance

A miner implementation claiming Seal-360 conformance MUST:

* publish a valid public encryption key,
* accept Encrypted Submissions over authenticated transport,
* decrypt submissions and recompute `template_hash`,
* reject submissions with mismatched hashes,
* process accepted transactions under standard Bitcoin consensus rules.

Miners MAY refuse submissions for any reason.

---

### 14.3 System Conformance

A system claiming Seal-360 conformance MUST:

* preserve public broadcast as a reachable execution option through explicit user or policy authorization,
* avoid introducing new trust assumptions,
* avoid modifying Bitcoin consensus, Script, or relay rules,
* ensure all failure modes surface execution state and defer any public broadcast to explicit authorization.

Any implementation that prevents user-authorized public broadcast or asserts miner guarantees is non-conformant.

---

## Annex A: Optional Hardening Extensions

Seal-360 MAY be extended with additional hardening layers that compose cleanly with the core protocol. These extensions are optional and do not modify Seal-360 conformance requirements.

None of the extensions below are required to resolve the BIP-360 execution window.

---

### A.1 Pre-Construction Authorization

Wallets MAY require external authorization before constructing a transaction intended for private relay.

Examples include:

* multi-party approval,
* policy-based spending limits,
* time-based execution windows.

Authorization MUST occur before signing.
Authorization MUST NOT modify public broadcast authorization semantics.

---

### A.2 Refusal-Based Execution Control

Wallets MAY implement refusal semantics that prevent execution under certain conditions, such as:

* policy violation,
* stale authorization,
* environment integrity failure.

Refusal MUST occur before private relay initiation.
Refusal MUST NOT suppress user-authorized public broadcast unless explicitly user-configured.

---

### A.3 Time-Bound Execution

Private relay MAY be constrained by explicit time windows:

* earliest submission time,
* latest acceptable confirmation time,
* bounded retry windows.

Time constraints MUST be enforced locally by the wallet.

---

### A.4 Multi-Party Approval

High-value or sensitive transactions MAY require:

* quorum approval,
* role-based signoff,
* delayed execution windows.

Such approvals apply to transaction construction and signing only.

---

### A.5 Audit and Telemetry (Local)

Wallets MAY record:

* submission attempts,
* miner responses,
* user-authorized public broadcast events,
* confirmation timing.

Audit data MUST remain local unless explicitly exported by the user.

---

### A.6 Non-Interference Rule

Optional hardening extensions MUST NOT:

* alter transaction bytes after signing,
* bypass template hash verification,
* remove or weaken user-authorized public broadcast,
* assert miner guarantees,
* modify Bitcoin consensus behavior.

Any extension that violates these constraints is non-conformant.

---

## Annex B: Comparative Analysis

### B.1 Standard Bitcoin (No Seal-360)

| Phase                   | Visibility       | Risk                                 |
| ----------------------- | ---------------- | ------------------------------------ |
| Transaction signing     | Local only       | None                                 |
| Public broadcast        | Global mempool   | Signatures publicly visible          |
| Pre-confirmation window | Global observers | Post-signing quantum attack possible |
| Confirmation            | On-chain         | Historical exposure only             |

**Result:** Execution window is fully exposed.

---

### B.2 BIP-360 Only

| Phase                   | Visibility       | Risk                           |
| ----------------------- | ---------------- | ------------------------------ |
| Output creation         | No key-path      | Output protected               |
| Transaction signing     | Local only       | None                           |
| Public broadcast        | Global mempool   | Signatures publicly visible    |
| Pre-confirmation window | Global observers | Execution window still exposed |
| Confirmation            | On-chain         | Historical exposure only       |

**Result:** Output-layer hardening only. Execution window remains exposed.

---

### B.3 BIP-360 + Seal-360

| Phase                   | Visibility          | Risk                       |
| ----------------------- | ------------------- | -------------------------- |
| Output creation         | No key-path         | Output protected           |
| Transaction signing     | Local only          | None                       |
| Private relay           | Selected miner only | Signatures encrypted       |
| Pre-confirmation window | Miner-only          | Public exposure eliminated |
| Confirmation            | On-chain            | Historical exposure only   |

**Result:** Output-layer and execution-layer exposure addressed.

---

### B.4 Security Impact Summary

| Property                         | Standard Bitcoin | BIP-360 | BIP-360 + Seal-360 (Quantum-Safety Mode) | BIP-360 + Seal-360 (Liveness Mode) |
|----------------------------------|------------------|---------|------------------------------------------|------------------------------------|
| Output quantum hardening         | No               | Yes     | Yes                                      | Yes                                |
| Execution-window protection      | No               | No      | Yes                                      | Opportunistic                      |
| Public mempool exposure          | Yes              | Yes     | No prior to confirmation                 | After authorization                |
| Payload confidentiality (PQ)    | No               | No      | Yes (EBT)                                | No                                 |
| Liveness guarantee               | Yes              | Yes     | Conditional                              | Yes                                |
| Consensus changes required       | No               | Yes     | Yes                                      | Yes                                |

Seal-360 in quantum-safety mode provides execution-window protection by removing
public mempool exposure and applying Encrypt-Before-Transport payload
confidentiality while private relay succeeds.

Seal-360 in liveness mode preserves standard Bitcoin execution behavior after
explicit public broadcast authorization, while still allowing opportunistic
reduction of execution-window exposure when private relay succeeds.

---

### B.5 Interpretation

Seal-360 does not replace Bitcoin broadcast.

It opportunistically reduces execution-window exposure when miners cooperate by deferring public mempool exposure during transaction execution.

When private relay does not succeed, users may explicitly authorize public broadcast, resulting in execution behavior equivalent to standard Bitcoin for that transaction. Such a transition may introduce additional delay or correlation metadata relative to immediate public broadcast, but it is never automatic and always requires explicit authorization.

Seal-360 therefore prioritizes user sovereignty over execution semantics while providing a conditional reduction in execution-window exposure when private relay succeeds.

---

## Annex C: Test Vectors and Conformance Tests (Reference)

This annex defines reference test vectors and required behavioral tests for
Seal-360 implementations.

Unless explicitly stated otherwise, tests in this annex are REQUIRED for any
implementation claiming Seal-360 conformance.

---

### C.1 Successful Private Relay (Happy Path)

**Inputs**

- Fully signed Taproot (SegWit v1) Bitcoin transaction
- Valid miner ML-KEM-768 public key
- Valid offer expiry (`expires_at` in the future)

**Process**

1. Wallet constructs and signs the transaction.
2. Wallet computes `template_hash`.
3. Wallet encrypts the transaction and submits via Encrypted Submission.
4. Miner decrypts and verifies AEAD integrity.
5. Miner recomputes and verifies `template_hash`.
6. Miner includes the transaction in a block.

**Expected Result**

- Transaction confirms on-chain.
- No public mempool exposure occurs prior to confirmation.
- Replay cache records (`miner_id`, `submission_id`) until expiry.

---

### C.2 Replay Detection

**Inputs**

- Valid Encrypted Submission reused verbatim.

**Process**

1. Submit encrypted payload to miner.
2. Submit the identical payload again before `expires_at`.

**Expected Result**

- Second submission is rejected.
- Miner reports replay detection.
- No execution state downgrade occurs.

---

### C.3 Expired Offer Rejection

**Inputs**

- Encrypted Submission where `expires_at` is in the past.

**Expected Result**

- Miner rejects submission prior to decryption.
- Wallet surfaces execution state.
- No public broadcast occurs without explicit authorization.

---

### C.4 AEAD Integrity Failure

**Inputs**

- Encrypted Submission with modified ciphertext or AAD field.

**Expected Result**

- AEAD decryption fails.
- Miner rejects submission.
- Wallet surfaces execution state.

---

### C.5 Template Hash Mismatch

**Inputs**

- Encrypted Submission where decrypted transaction bytes differ from
  supplied `template_hash`.

**Expected Result**

- Miner recomputes `template_hash`.
- Mismatch detected.
- Submission rejected as a hard integrity failure.

---

### C.6 Witness Malleability Rejection

**Inputs**

- Non-Taproot transaction (legacy or SegWit v0).

**Expected Result**

- Wallet refuses to initiate Seal-360 execution.
- Transaction MAY be broadcast only via explicit public authorization.

---

### C.7 Transport Failure

**Inputs**

- Valid encrypted payload.
- Transport failure or endpoint unreachable.

**Expected Result**

- Wallet retries alternative transports or miners if configured.
- Execution state is surfaced after retry exhaustion.
- No automatic public broadcast occurs.

---

### C.8 Network Partition Handling

**Inputs**

- Valid private relay submission.
- Miner includes transaction during wallet network partition.

**Process**

1. Wallet loses confirmation visibility.
2. Timeout expires.
3. Wallet checks mempool and chain state before authorization.

**Expected Result**

- Wallet detects confirmation or ambiguity.
- Wallet MUST NOT authorize public broadcast without explicit user or policy
  confirmation.

---

### C.9 Correlation Downgrade Guard

**Inputs**

- Failed private relay followed by authorized public broadcast.

**Process**

1. Wallet enforces delay and fee normalization rules.
2. Wallet broadcasts transaction only after mitigation steps.

**Expected Result**

- No immediate broadcast correlation.
- Transaction broadcast occurs only after explicit authorization.

---

### C.10 RNG Failure Handling

**Inputs**

- Simulated failure of cryptographically secure RNG during nonce generation.

**Expected Result**

- Wallet aborts Encrypted Submission.
- Execution state is surfaced.
- No encrypted payload is transmitted.

---

### C.11 Miner Restart Replay Safety

**Inputs**

- Valid submission processed by miner.
- Miner restarts.
- Identical submission resent.

**Expected Result**

- Replay cache persists across restart.
- Submission is rejected.
- Miner remains conformant.

---

### C.12 Conformance Summary

A conformant Seal-360 implementation MUST demonstrate:

- strict enforcement of Encrypt-Before-Transport semantics,
- replay detection persistence,
- Taproot-only enforcement,
- no silent downgrade to public broadcast,
- correct handling of partitions and transport failure,
- explicit authorization for all public broadcasts.

Failure to pass any REQUIRED test above constitutes non-conformance.

---

## Annex D: Deployment Scenarios

### D.1 Standalone Bitcoin Wallet

**Context:** Consumer or self-custody wallet adding execution-layer hardening.

**Deployment:**

* Wallet implements miner discovery and Encrypted Submission.
* Wallet enforces timeout-based execution state surfacing during private relay.
* User or policy explicitly authorizes public broadcast if private relay does not succeed.
* No other post-quantum components are required.

**Properties:**

* Reduced execution-window exposure when private relay succeeds.
* No automatic mempool exposure.
* No change to custody model.
* Minimal additional complexity.

---

### D.2 Mining Pool Relay Endpoint

**Context:** Mining pool offering private relay as a service.

**Deployment:**

* Pool publishes encryption key and submission endpoint.
* Pool verifies `template_hash` before inclusion.
* Pool includes transactions per normal fee policy.

**Properties:**

* Optional service, no obligation to include.
* No custody or authorization role.
* No protocol lock-in.

---

### D.3 Institutional or OTC Desk

**Context:** High-value settlement requiring minimized execution exposure.

**Deployment:**

* Wallet defaults to private relay.
* Multiple miners queried for redundancy.
* Strict timeout and audit logging enabled.

**Properties:**

* Reduced exposure during negotiation and execution.
* Preserves censorship resistance.
* Compatible with existing Bitcoin infrastructure.

---

### D.4 Mixed Environment (Liveness-Priority Configuration)

**Context:** Environments with unreliable or limited miner support where confirmation liveness is prioritized.

**Deployment:**

* Wallet attempts private relay opportunistically.
* On private relay failure, execution state is surfaced.
* User or policy explicitly authorizes public broadcast when liveness is required.

**Properties:**
- Public broadcast is performed only after explicit authorization.
- Execution semantics revert to standard Bitcoin behavior once broadcast
  is authorized.
- Opportunistic execution-window protection applies when private relay
  succeeds.
- Failed private relay followed by public broadcast may introduce additional
  delay or correlation relative to immediate broadcast.

---

### D.5 Post-Quantum Hardened Stack (Optional)

**Context:** Deployments combining Seal-360 with broader post-quantum controls.

**Deployment:**

* Seal-360 used for execution delivery.
* Other systems may gate signing or construction.
* Public broadcast, if used, requires explicit user or policy authorization.

**Properties:**

* Layered defense without protocol coupling.
* Seal-360 remains execution-only.
* No authority overlap or conflict.

---

## Annex E: Frequently Asked Questions

**Do I need BPC to use Seal-360?**  
No. Seal-360 is a standalone execution-layer protocol. It does not require Bitcoin Pre-Contracts.

**Does Seal-360 replace public broadcast?**  
No. Public broadcast remains available as an execution option. Seal-360 defers public broadcast and requires explicit user or policy authorization before any transaction is submitted to the public mempool.

**Does Seal-360 require miner trust?**  
No. Miners are untrusted. Miner refusal, delay, or failure does not grant miners execution authority and does not trigger automatic public broadcast. Execution state is surfaced and any public broadcast requires explicit authorization.

**Is Seal-360 censorship resistant?**  
Seal-360 preserves Bitcoin’s censorship resistance by ensuring that user-authorized public broadcast is always available. It does not guarantee inclusion during private relay and does not automatically submit transactions to the public network.

**Does Seal-360 provide anonymity?**  
No. Seal-360 reduces the number of observers during execution but does not provide anonymity or metadata-privacy guarantees.

**Is Seal-360 quantum-safe?**  
Seal-360 provides quantum-resistant payload confidentiality
(Encrypt-Before-Transport, EBT) during the execution window when the recommended
cryptographic profile is used.

It does not modify Bitcoin’s on-chain signature scheme, does not provide
post-quantum signatures on-chain, and does not claim end-to-end quantum safety.

Seal-360 reduces a specific execution-window attack surface and complements
BIP-360’s output-layer hardening; it is not a complete quantum-security solution
by itself.


**Can Seal-360 be used with legacy outputs?**  
Yes. Seal-360 can be used with any broadcast-valid Bitcoin transaction. Output-layer quantum hardening requires BIP-360.

**What happens if all miners refuse?**  
The wallet surfaces execution state and awaits user or policy authorization. The user may retry with different miners, wait and retry later, or explicitly authorize public broadcast to proceed under standard Bitcoin assumptions.

**Can a miner modify the transaction?**  
No. Any modification invalidates the template hash and MUST be rejected.

**Is Seal-360 compatible with RBF and CPFP?**  
Yes. Standard Bitcoin RBF and CPFP semantics apply.

**Where are common design objections addressed (trust, centralisation, on-chain, adoption)?**  
See Section 7.7.

---

## Annex F: Wallet UX Guidelines and Miner Selection Interface (Informative)

This annex provides non-normative guidance for wallet developers implementing user interfaces for Seal-360.

It illustrates common patterns for miner discovery, fee presentation, public broadcast authorization handling, and user choice, while emphasizing that all UI and UX decisions remain implementation-defined.

Nothing in this annex defines protocol behavior, security requirements, message formats, or conformance criteria.

---

### F.1 Purpose

Seal-360 introduces an execution path in which transactions may be delivered privately to miners using quantum-resistant encryption prior to confirmation.

This annex exists to:

* illustrate practical wallet UX patterns for private relay
* clarify user sovereignty in miner selection
* encourage transparent fee presentation
* document common public broadcast authorization and error-handling behavior

Wallet developers may implement these patterns partially, differently, or not at all.

In this annex, references to public broadcast are colloquial and refer exclusively to explicit user-authorized public broadcast. No automatic protocol behavior is implied.

---

### F.2 Tier Naming and Scope Clarification

The tier names used in this annex — Transactional, Baseline, and Enterprise — mirror PQHD custody tier terminology for conceptual consistency only.

In Seal-360, these tiers describe user-interface complexity and configuration depth, not custody guarantees, key management, quorum requirements, or PQHD conformance.

No Seal-360 UX tier implies the use of PQHD.

---

### F.3 Core UX Principles

#### F.3.1 Sovereignty Preservation

Wallets should preserve user sovereignty in miner selection and delivery method:

* users may be presented with explicit miner choices
* users may delegate selection to wallet defaults
* wallets should not obscure the distinction between private relay and public broadcast

Users should always be able to understand how their transaction will be delivered.

---

#### F.3.2 Transparency

When presenting private relay options, wallets should clearly communicate:

* fee terms in sat/vB
* any fee premium relative to public broadcast
* offer expiry windows, if short-lived
* whether delivery is via private relay or public broadcast

Miner identity may be shown explicitly or abstracted based on privacy goals.

---

#### F.3.3 Simplicity Gradient

Wallets should adapt miner-selection UX to their intended audience:

* Transactional (UX Tier): Minimal interface, legacy-compatible, optional private relay
* Baseline (UX Tier): Default private relay with limited configuration
* Enterprise (UX Tier): Policy-driven selection, dashboards, auditability

No tier is required. All are optional design choices.

---

### F.4 Recommended Interface Patterns

The following patterns are illustrative only.

#### F.4.1 Transactional (UX Tier) — Minimal Toggle

```
┌─────────────────────────────────────┐
│ Transaction Details                 │
├─────────────────────────────────────┤
│ Amount:          1.5 BTC            │
│ Recipient:       bc1q...xyz         │
│                                     │
│ [ ] Private Relay                   │
│     (Experimental)                  │
│                                     │
│ Estimated Fee:   ~12 sat/vB         │
│ Estimated Time:  3–6 blocks         │
│                                     │
│ [        Send Transaction      ]    │
└─────────────────────────────────────┘
```

---

#### F.4.2 Baseline (UX Tier) — Single-Toggle Pattern

```

┌─────────────────────────────────────┐
│ Transaction Details                 │
├─────────────────────────────────────┤
│ Amount:          1.5 BTC            │
│ Recipient:       bc1q...xyz         │
│                                     │
│ [x] Private Relay (user-selected)   │
│     (No Mempool Exposure)           │
│     Encrypted delivery to miner     │
│     +2.5 sat/vB premium             │
│                                     │
│ Estimated Fee:   15.5 sat/vB        │
│ Estimated Time:  Next block         │
│                                     │
│ [       Review and Send        ]    │
└─────────────────────────────────────┘

```
---

#### F.4.3 Enterprise (UX Tier) — Advanced Dashboard

```

┌─────────────────────────────────────────────────────┐
│ Miner Selection Dashboard                            │
├─────────────────────────────────────────────────────┤
│                                                     │
│ Public Broadcast (Standard)                          │
│ Estimated: 12 sat/vB  3–6 blocks                     │
│ [Use This Method]                                    │
│                                                     │
│ Private Relay                                        │
│ ┌──────────────┬──────┬─────────┬──────┬──────┐     │
│ │ Pool         │ Fee  │ Latency │ Rel. │ Sel. │     │
│ ├──────────────┼──────┼─────────┼──────┼──────┤     │
│ │ ● Braiins    │ 14.2 │ 85ms    │ 99.8%│  ✓   │     │
│ │ ○ Foundry    │ 16.0 │ 120ms   │ 99.9%│      │     │
│ │ ○ ViaBTC     │ 15.5 │ 210ms   │ 98.7%│      │     │
│ │ ○ Luxor      │ 17.1 │ 95ms    │ 99.5%│      │     │
│ └──────────────┴──────┴─────────┴──────┴──────┘     │
│                                                     │
│ Policy Options:                                     │
│ [ ] Prefer non-censoring jurisdictions               │
│ [ ] Require ≥99% historical inclusion                │
│ [x] Require explicit authorization before            │
│     public broadcast                                 │
│                                                     │
│ [        Send Securely                         ]     │
└─────────────────────────────────────────────────────┘

```
---

### F.5 Fee Display Standards

Wallets should display:

* fee rate in sat/vB
* total fee in sats
* estimated confirmation priority
* offer expiry time, if short

Comparative display example:

```
Private relay:     15 sat/vB  next block
Public broadcast:  12 sat/vB  3–6 blocks
Difference:        +3 sat/vB  +25%
```

Wallets should consider:

* refreshing stale offers
* warning on large fee changes
* re-querying if user delays sending

---

### F.6 Privacy Considerations

Wallets should consider:

* querying multiple miners in parallel
* randomizing timing and ordering
* using privacy-preserving transports

Wallets may:

* display miner aliases
* omit jurisdictional metadata
* avoid persistent storage of selection history

---

### F.7 Error States and Public Broadcast Authorization UX

No private relay endpoints available:

```
╭──────────────────────────────────╮
│ No Private Relay Available       │
├──────────────────────────────────┤
│                                  │
│ All participating miners are     │
│ currently unavailable.           │
│                                  │
│ [ Use Public Broadcast ]         │
│ [ Cancel Transaction ]           │
╰──────────────────────────────────╯
```

Offer expiry during composition:

```
Miner offer expired. Re-fetch?
[Get New Quotes] [Continue Public]
```

Wallets may surface execution state after timeout and require explicit user or policy authorization before any public broadcast.

---

### F.8 Accessibility Considerations

Wallets should ensure:

* keyboard navigability
* screen-reader clarity for fee differences
* sufficient contrast
* consistent iconography

---

### F.9 Implementation Checklist

Wallets implementing this annex's UX patterns should:

* explain private relay clearly
* display fee premiums transparently
* implement graceful handling of user-authorized public broadcast
* preserve user sovereignty
* handle errors with clear guidance
* meet platform accessibility standards

---

### F.10 Informative Status

This annex is informative only.

Wallet developers may implement interfaces differently and must comply with normative requirements in the core specification.

---

## Annex G: Terminology

### Private Relay

The complete Seal-360 execution path, including miner discovery, transaction construction, Encrypted Direct-to-Miner Submission, miner processing, and an explicit execution decision. Private relay is the defining characteristic of Seal-360 and governs how a signed transaction may be delivered privately prior to confirmation.

Private relay does **not** include automatic public broadcast. Public broadcast, when used, is an explicitly authorized execution choice outside the private relay path.

---

### Encrypt-Before-Transport (EBT)

A security invariant requiring that sensitive payloads are cryptographically
encrypted prior to entering any transport, framing, routing, or delivery
mechanism.

EBT defines payload confidentiality only and does not define transport,
authentication, routing, retry, or delivery semantics.

Encrypt-Before-Transport (EBT) is a shared security invariant across the broader
post-quantum (PQ) stack, including PQHD, PQSF, and related execution and policy
frameworks. Seal-360 defines and enforces EBT locally for execution-layer
correctness and does not require adoption of other stack components for
conformance.

---

### Encrypted Direct-to-Miner Submission (Encrypted Submission)

The act of encrypting a fully signed Bitcoin transaction under a miner’s ML-KEM-768 public key and transmitting it directly to that miner for block inclusion.

This is the core protocol action within private relay and is the mechanism by which public mempool exposure is avoided during execution.

---

### Template Hash

A SHA-256 cryptographic commitment to the complete signed Bitcoin transaction bytes, computed after signing.

The template hash binds the encrypted payload to its exact transaction instance and prevents miners from modifying the transaction after decryption without detection.

```
template_hash = SHA256(raw_tx_bytes)
```

Where:

* `raw_tx_bytes` is the exact Bitcoin wire serialization of the transaction that would be broadcast if public submission is later authorized.
* Serialization includes all inputs, outputs, scripts, witness data (if present), and locktime.

Any modification to the transaction bytes after signing MUST produce a different template hash and MUST result in rejection by the miner or refusal by the wallet.

---

### Public Broadcast

The act of submitting a fully signed Bitcoin transaction to the standard public peer-to-peer network and mempool.

Public broadcast is always available to the user or policy engine but is **never triggered automatically** by the Seal-360 protocol. When authorized, public broadcast re-enters the standard Bitcoin threat model and is considered quantum-unsafe during the execution window.

---

### Execution Window

The time period between transaction signing and blockchain confirmation during which transaction signatures may become observable.

In standard Bitcoin execution, this window typically spans one block interval (~10 minutes) and involves public mempool exposure. Seal-360 eliminates public mempool exposure during this window while private relay succeeds.

---

### Quantum-Resistant

Cryptographic algorithms designed to resist attacks by quantum computers.

Seal-360 uses ML-KEM-768, a NIST-standardized post-quantum key encapsulation mechanism, to protect transaction confidentiality during Encrypted Direct-to-Miner Submission.

---

### Miner Discovery

The process by which wallets locate, authenticate, and select miners that support Seal-360 private relay.

Miner discovery mechanisms and requirements are defined normatively in Annex H.

---

### Template Hash Binding

The cryptographic property that binds an encrypted submission to a specific signed Bitcoin transaction instance.

Any post-signing modification to the transaction bytes invalidates the template hash, causing rejection by the miner or refusal by the wallet. Template hash binding ensures miners gain visibility but not control over transaction execution.


---

## Annex H: Miner Discovery Protocol (Normative)

This annex defines a minimum interoperable discovery mechanism. Alternative discovery mechanisms MAY be used provided they meet the authentication and key-integrity requirements defined herein.

---

### H.1 Discovery Mechanisms

Wallets discover Seal-360-capable miners via one of the following authenticated mechanisms.

All discovery mechanisms MUST provide an authenticated binding between a miner identity and its advertised public encryption keys.

#### H.1.1 HTTPS Manifest Discovery (RECOMMENDED)

**Endpoint:**  
`https://miner.example/.well-known/seal360.json`

**Authentication:**
- TLS certificate validates endpoint identity
- Certificate MUST be valid and trusted
- TLS 1.3 or later REQUIRED
- HSTS SHOULD be enabled

The manifest format is defined in Section H.2.

#### H.1.2 DNS-Based Discovery (ALTERNATIVE)

**TXT Record:**  
`_seal360.miner.example`

**Format:**
```text
_seal360.miner.example. 3600 IN TXT "v=seal360-1 manifest=https://miner.example/.well-known/seal360.json"
```

#### H.1.3 DHT-Based Discovery (EXPERIMENTAL)

**Status:** Experimental. Not recommended for production use.

**Use case:** Censorship-resistant environments only.

**Requirements:**
- Requires an additional trust anchor
- Out of scope for baseline Seal-360 deployments

---

### H.2 Manifest Format

Miners MUST publish a signed manifest in JSON format describing their Seal-360 private relay endpoints and encryption keys.

The manifest communicates execution terms only and MUST NOT be interpreted as an inclusion guarantee.

```json
{
  "version": "1.0",
  "miner_id": "example-pool-01",
  "endpoints": [
    {
      "url": "https://relay.example.com/submit",
      "ml_kem_768_pubkey": "base64-encoded-public-key",
      "key_id": "2026-01-key-01",
      "fee_terms": {
        "base_sat_vb": 12.5,
        "priority_multiplier": 1.5
      },
      "max_tx_size": 100000,
      "expires_at": "2026-02-15T00:00:00Z"
    }
  ],
  "identity_key": "base64-encoded-long-lived-key",
  "signature": "base64-encoded-signature",
  "published_at": "2026-01-15T00:00:00Z"
}
```

**Required Fields:**

* `version`: Protocol version (currently `"1.0"`)
* `miner_id`: Unique, stable identifier for the miner or pool
* `endpoints`: One or more Encrypted Submission endpoints
* `identity_key`: Long-lived public key used to sign the manifest
* `signature`: Signature over the canonicalized manifest
* `published_at`: ISO 8601 timestamp indicating publication time

**Endpoint Fields:**

* `url`: HTTPS endpoint for Encrypted Direct-to-Miner Submission
* `ml_kem_768_pubkey`: Public key used for ML-KEM-768 encryption
* `key_id`: Identifier for key rotation and selection
* `fee_terms`: Fee structure applicable to private relay
* `max_tx_size`: Maximum accepted transaction size in bytes
* `expires_at`: Offer expiry time (ISO 8601)

Manifests MAY include additional fields, but wallets MUST ignore unknown fields.

---

### H.3 Signature Verification

Wallets MUST verify manifest signatures before using any advertised Seal-360 endpoint.

**Signature Algorithm:** Ed25519

**Signing Process:**

1. Remove the `signature` field from the manifest.
2. Canonicalize the remaining JSON using RFC 8785 (JCS).
3. Compute the signature over the canonical bytes using the miner’s `identity_key`.
4. Encode the signature as base64 and populate the `signature` field.

**Verification Process:**

1. Fetch the manifest via an authenticated discovery mechanism.
2. Extract `identity_key` and `signature`.
3. Remove the `signature` field.
4. Canonicalize the remaining JSON using RFC 8785 (JCS).
5. Verify the Ed25519 signature using `identity_key`.
6. Reject the manifest if verification fails.

**Trust Anchoring:**

Manifest signatures provide integrity and key-rotation continuity, not identity by themselves. Identity is anchored by:

* TLS certificate validation for HTTPS discovery,
* DNSSEC validation for DNS-based discovery,
* External reputation systems or transparency logs where applicable.

Failure to verify a manifest signature MUST result in rejection of all associated endpoints.

---

## H.4 Key Rotation (Normative)

Miners claiming Seal-360 conformance MUST support encryption key rotation to
allow routine operational updates and emergency replacement.

Key rotation applies exclusively to the miner’s post-quantum encryption keys
used for Encrypted Direct-to-Miner Submission and MUST NOT alter transaction
execution semantics, authorization requirements, or public broadcast behavior.

---

### H.4.1 Key Identification

Each published encryption key MUST:

* be an ML-KEM-768 public key as defined in FIPS 203,
* be uniquely identified by a stable `key_id`,
* be bound to a specific validity window via `expires_at`,
* appear only within a signed discovery manifest (Section H.2).

Keys without a `key_id` or expiry information MUST be rejected.

---

### H.4.2 Scheduled Rotation

For planned key rotation, miners SHOULD follow the procedure below.

At least **24 hours** before the intended rotation time, the miner SHOULD:

* publish a new ML-KEM-768 public key in the discovery manifest,
* assign a new, unique `key_id` to the new key,
* include both the old and new keys in the manifest concurrently,
* set `expires_at` on the old key to a time no more than 24 hours in the future.

During this overlap window:

* miners MUST accept Encrypted Submissions encrypted to either key,
* wallets MUST prefer the newest non-expired key,
* new submissions SHOULD use the new key where available.

Extended overlap windows increase exposure if an older key is compromised and
SHOULD be avoided for high-value transactions.

---

### H.4.3 Post-Rotation Enforcement

After a key’s `expires_at` time:

* the expired key MUST be removed from the discovery manifest,
* Encrypted Submissions encrypted to the expired key MUST be rejected,
* replay state associated with the expired key MAY be safely discarded after
  expiry.

Acceptance of submissions encrypted to expired keys constitutes
non-conformance.

---

### H.4.4 Emergency Rotation

If compromise of an encryption key is suspected or confirmed, miners MAY perform
emergency rotation without advance notice.

In an emergency rotation:

* the compromised key MUST be immediately removed or marked invalid,
* Encrypted Submissions encrypted to the compromised key MUST be rejected,
* a replacement key SHOULD be published as soon as practicable.

Wallets encountering submission failure due to key invalidation:

* MUST re-fetch the discovery manifest,
* MUST retry key selection using a non-expired key if available,
* MUST NOT trigger automatic public broadcast as a result of key rotation.

Emergency rotation MUST NOT result in silent downgrade to public broadcast.

---

### H.4.5 Wallet Requirements

Wallets implementing Seal-360 MUST:

* validate key expiry using RFC 3339 UTC timestamps (Section 6),
* prefer the newest non-expired key when multiple keys are present,
* reject keys that are expired, revoked, or malformed,
* treat key-rotation-related submission failure as an execution failure and
  surface execution state accordingly.

Wallets MUST NOT continue using older keys when a newer valid key is available,
except during the defined overlap window.

---

### H.4.6 Scope Clarification

Key rotation affects confidentiality only.

Key rotation:

* does NOT alter transaction validity,
* does NOT alter authorization requirements,
* does NOT grant miners execution authority,
* does NOT trigger public broadcast.

All public broadcast decisions remain subject to Explicit Public Broadcast
Authorization (Section 11.4).

---

### H.5 Miner Discovery Hardening

Seal-360 miner discovery MUST resist Sybil attacks, identity churn, and
misrepresentation of execution capability.

Discovery mechanisms MUST NOT assume that advertised miner identities are
honest, unique, or economically independent.

---

#### H.5.1 Identity Stability Requirement

A miner identity (`miner_id`) MUST be stable across manifest updates.

Wallets MUST reject discovery entries where:

- `miner_id` changes without continuity of `identity_key`,
- `identity_key` rotates without a documented key-rotation event,
- multiple distinct endpoints advertise the same `miner_id` with conflicting
  parameters.

Frequent identity churn MUST be treated as a negative signal.

---

#### H.5.2 Hashpower Correlation Requirement

Wallets MUST require evidence that a discovered miner identity corresponds to
active Bitcoin block production.

At minimum, wallets MUST verify one of the following within a configurable
observation window:

- recent blocks mined that can be attributed to the miner or pool,
- publicly verifiable pool attribution (coinbase tags, payout addresses),
- an externally verifiable reputation registry that maps `miner_id` to observed
  hashpower.

Discovery entries lacking verifiable block production evidence MUST NOT be used
for private relay.

---

#### H.5.3 Minimum Activity Threshold

Wallets MUST enforce a minimum activity threshold before using a miner for
private relay.

The threshold MUST be based on one or more of:

- minimum blocks mined within the observation window,
- minimum share of recent network hashpower,
- minimum successful confirmation rate observed by the wallet.

The exact threshold is implementation-defined but MUST be documented.

---

#### H.5.4 Sybil Set Limitation

Wallets MUST limit the number of miners selected from any single identity or
attribution cluster.

Miners sharing:

- payout infrastructure,
- coinbase tagging patterns,
- `identity_key` lineage,
- network endpoints,

MUST be treated as a single cluster for selection and routing purposes.

Wallets MUST NOT route all private relay attempts exclusively to a single
cluster when multiple eligible clusters exist.

---

#### H.5.5 Offer Cross-Validation

Wallets MUST cross-validate miner offers when multiple candidates are available.

At minimum, wallets MUST compare:

- fee premiums,
- expiry windows,
- reliability signals.

If all discovered miners advertise identical or near-identical premium terms,
wallets SHOULD surface a cartel-risk warning to the user or policy engine.

---

#### H.5.6 Failure Handling

Failure to satisfy any requirement in this section MUST result in:

- exclusion of the miner from private relay selection, and
- surfacing of execution state to the user or policy engine.

Miner discovery hardening failures MUST NOT trigger automatic public broadcast.
Public broadcast remains subject to Explicit Public Broadcast Authorization
(Section 11.4).

---

#### H.5.7 Scope Clarification

This section defines minimum discovery hardening requirements.

It does not:

- guarantee miner honesty,
- prevent collusion,
- prevent economic coercion.

It reduces trivial Sybil and impersonation attacks while preserving deployability
and incremental adoption.

---

### H.6 Caching and Refresh

Wallets SHOULD cache validated miner manifests to reduce discovery latency and network load.

**Cache Duration:**

* Manifests SHOULD be cached until their earliest `expires_at`.
* Minimum cache duration SHOULD be 1 hour.
* Maximum cache duration SHOULD NOT exceed 7 days.

**Refresh Triggers:**
Wallets SHOULD refresh cached manifests when any of the following occur:

* Encrypted Submission fails due to authentication or key errors.
* The cached manifest is within 1 hour of expiry.
* The user or policy engine explicitly requests a refresh.
* Key rotation is detected.

**Failure Handling:**

* If manifest refresh fails, wallets MAY continue using a cached manifest if it is not expired.
* If the cached manifest is expired and refresh fails, wallets MUST surface the execution state to the user or policy engine.

Caching MUST NOT suppress user-authorized public broadcast or delay execution beyond wallet-defined timeouts.

---

### H.7 Privacy Considerations

Seal-360 reduces execution visibility but does not provide anonymity.

Wallets SHOULD adopt basic privacy-preserving practices when performing miner discovery and Encrypted Submission.

**Discovery Privacy:**

* Query multiple miners in parallel where possible.
* Randomize query ordering and timing.
* Avoid persistent miner preferences across sessions.

**Transport Privacy:**

* Use ephemeral connections.
* Avoid long-lived or pinned submission sessions.
* Use Tor or VPN transports where available.

**Metadata Minimization:**

* Avoid logging miner selection history by default.
* Avoid correlating miner identity with user identity.
* Avoid transmitting unnecessary client metadata.

Private relay failure does not automatically imply public broadcast.  
Public broadcast occurs only after explicit user or policy authorization.

These measures reduce trivial correlation but do not eliminate metadata leakage.

---

### H.8 Security Considerations

Seal-360 discovery and submission infrastructure MUST be treated as adversarial.

**TLS Requirements:**

* TLS 1.3 or later REQUIRED.
* Strong cipher suites only.
* Certificate validation MUST be enforced.
* Certificate pinning MAY be used but MUST support rotation.

**Manifest Validation:**

* Reject manifests with invalid signatures.
* Reject manifests with expired endpoint `expires_at`.
* Reject manifests missing required fields.
* Reject malformed or non-canonical JSON.

**Key Validation:**

* Verify ML-KEM-768 public key encoding.
* Reject keys outside valid parameter space.
* Reject duplicate or conflicting `key_id` values.

Failure of any validation step MUST result in refusal of private relay and surfacing of execution state to the user or policy engine.

No validation failure may trigger automatic public broadcast. Public broadcast MAY occur only after explicit user or policy authorization in accordance with Section 11.

---

### H.9 Error Handling

Miners MAY return structured error responses to Encrypted Submissions.

Wallets MUST treat all errors as non-fatal and MUST preserve user sovereignty over execution decisions.

**Example Errors:**

Invalid manifest:
```json
{
  "error": "invalid_manifest",
  "reason": "Signature verification failed"
}
```

Expired offer:

```json
{
  "error": "offer_expired",
  "reason": "Offer expired at 2026-01-15T00:00:00Z"
}
```

Service unavailable:

```json
{
  "error": "service_unavailable",
  "reason": "Temporarily not accepting submissions"
}
```

On any error:

* Wallets MAY retry Encrypted Submission with a different miner.
* Wallets MUST surface execution state to the user or policy engine.
* Wallets MUST NOT automatically broadcast the transaction to the public mempool.
* Public broadcast MAY occur only after explicit user or policy authorization in accordance with Section 11.

No error condition may result in silent drop, automatic broadcast, or indefinite execution deadlock.

---

### H.10 Conformance

Miners claiming Seal-360 conformance MUST:

* Publish discovery manifests via an authenticated mechanism.
* Sign manifests with Ed25519.
* Publish valid ML-KEM-768 public encryption keys.
* Implement key rotation per Section H.4.
* Accept Encrypted Submissions at advertised endpoints.
* Verify `template_hash` after decryption.
* Reject submissions with mismatched hashes.

Wallets claiming Seal-360 conformance MUST:

* Verify manifest signatures.
* Verify miner public key authenticity.
* Enforce key rotation semantics.
* Implement caching and refresh rules.
* Enforce bounded retry and timeout behavior during private relay.
* Surface execution state and failure reasons to the user or policy engine.
* Require explicit user or policy authorization before any public broadcast.

Failure to meet any requirement above constitutes non-conformance.

---

## Annex I: Optional Stack Integration (Informative)

This annex describes how Seal-360 MAY be integrated with broader post-quantum (PQ) components for deployments requiring additional hardening, monitoring, or policy control.

**Status:** Informative only. Nothing in this annex is required for Seal-360 conformance.

Seal-360 is designed to operate as a standalone, Bitcoin-native execution-layer protocol. All integrations described here are optional and MUST NOT alter Seal-360’s core guarantees, user-authorized public broadcast semantics, or consensus compatibility.

This annex exists for operators deploying layered post-quantum security stacks and may be ignored entirely by standalone Seal-360 implementations.

---

### I.1 Purpose and Scope

Seal-360 requires no external dependencies beyond BIP-360-compliant Bitcoin transaction handling and standard cryptographic libraries.

Organizations deploying broader post-quantum security architectures MAY integrate Seal-360 with complementary components to obtain additional guarantees such as:

- deterministic canonical encoding,
- verifiable time semantics,
- execution-window exposure detection,
- policy-driven execution control.

Nothing in this annex modifies Seal-360’s core protocol, threat model, or conformance requirements.

---

### I.2 Integration Philosophy

Seal-360 remains correct, safe, and deployable without any stack integration.

Optional integrations provide additional assurance, not additional authority.

| Property                                | Seal-360 Alone | With Optional Integration |
|-----------------------------------------|----------------|---------------------------|
| Eliminates public mempool exposure      | Yes            | Yes                       |
| User-authorized public broadcast        | Yes            | Yes                       |
| Consensus compatibility (BIP-360)       | Yes            | Yes                       |
| Payload confidentiality (EBT)           | Yes            | Yes                       |
| Canonical encoding                      | Not provided   | Provided                  |
| Verifiable time binding                 | Not provided   | Provided                  |
| Leak detection                          | Not provided   | Provided                  |
| Policy enforcement                     | Not provided   | Provided                  |

Stack integration MUST NOT:
- trigger public broadcast automatically,
- alter signed transaction bytes,
- assert miner guarantees,
- introduce new trust assumptions,
- modify Bitcoin consensus behavior.

Public broadcast, if used, remains an explicitly authorized execution choice
outside the Seal-360 private relay mechanism.

Encrypt-Before-Transport (EBT) is used consistently across the PQ stack to ensure
that sensitive payloads are encrypted prior to entering any transport or routing
layer. Seal-360 adopts this invariant independently and does not introduce any
normative dependency on other PQ components.

---

### I.3 Optional Canonical Encoding (PQSF)

**Component:** Post-Quantum Serialization Format (PQSF)

PQSF MAY be used to provide deterministic encoding and algorithm agility for Seal-360 message payloads.

When PQSF is used, Encrypted Submission packages MAY be encoded using Deterministic CBOR instead of JSON.

```

EncryptedSubmissionPackage = {
version: tstr,
kem_ct: bstr,
nonce: bstr(24),
ciphertext: bstr,
template_hash: bstr(32),
key_id: tstr,
submission_id: bstr(16)
}

```

**Benefits:**
- Eliminates serialization ambiguity
- Enables cryptographic suite indirection
- Improves cross-component determinism

**Trade-offs:**
- Additional dependency
- Less human-readable
- Not required for baseline interoperability

PQSF integration is OPTIONAL and not required for Seal-360 conformance.

---

### I.4 Optional Enhanced AAD Binding

Seal-360 requires `template_hash` to be included as AEAD associated data.

Deployments MAY extend AAD to include additional context, such as:

- submission identifiers,
- miner identifiers,
- key identifiers,
- expiry information.

Baseline (conformant):
```

aad = template_hash

```

Extended (optional):
```

aad = canonical_encode({
template_hash,
submission_id,
miner_id,
key_id,
expiry
})

```

Extended AAD binding prevents replay, substitution, and cross-endpoint confusion but MUST NOT be required for interoperability.

---

### I.5 Optional Verifiable Time Binding (Epoch Clock)

Deployments MAY integrate verifiable time sources to bind submission expiry and replay semantics.

**Seal-360 alone:**
- Uses wall-clock or ISO 8601 timestamps
- Wallet-enforced expiry only

**With verifiable time binding:**
- Signed or anchored time artifacts
- Deterministic replay protection
- Long-term auditability

Time binding is OPTIONAL and MUST NOT trigger public broadcast automatically or suppress user-authorized public broadcast.

---

### I.6 Optional Policy Enforcement (PQSEC)

Policy frameworks MAY be layered above Seal-360 to guide behavior such as:

- miner selection preferences,
- retry strategies,
- public broadcast authorization timing,
- RBF escalation.

Policy enforcement MUST NOT:
- override user-authorized public broadcast,
- prevent public broadcast indefinitely,
- remove user sovereignty.

Policy layers influence execution decisions but do not redefine Seal-360 semantics.

---

### I.7 Optional Execution Coordination (PQEH)

Execution coordination frameworks MAY orchestrate when transactions are permitted to enter the execution window.

Seal-360 defines how transactions are delivered.
Coordination frameworks define when delivery is permitted.

Both layers compose cleanly but remain independent.

---

### I.8 Integration Safety Rules

Any optional integration MUST preserve:

- transaction byte integrity after signing,
- user-authorized public broadcast,
- miner non-trust assumptions,
- Bitcoin consensus compatibility.

Optional integrations MUST NOT:

- trigger public broadcast automatically,
- suppress or block user-authorized public broadcast,
- introduce execution deadlock,
- alter signed transaction bytes,
- assert miner guarantees,
- modify Bitcoin consensus behavior beyond that defined by BIP-360.

Any integration that violates these constraints is non-conformant with Seal-360.

---

### I.9 Summary of Optional Integrations

Seal-360 is complete and correct on its own.

Optional integrations add:
- stronger determinism,
- better monitoring,
- richer policy control.

They are strictly additive and MUST NOT be required for baseline deployment or conformance.

---

### I.11 Conformance and Stack Integration

Seal-360 conformance does NOT require stack integration.

A wallet or miner claiming Seal-360 conformance MAY:
- use JSON instead of PQSF,
- use Unix timestamps instead of verifiable time,
- implement simple leak detection instead of ZEB,
- omit policy frameworks,
- omit execution coordination.

All of these remain fully conformant implementations.

Stack integration is:
- optional for all deployments,
- recommended for comprehensive PQ frameworks,
- not required for Bitcoin ecosystem adoption.

---

### I.12 Migration Path

Incremental adoption strategy:

**Phase 1: Seal-360 Standalone**
- Wallet implements private relay
- Miner publishes ML-KEM-768 key
- JSON-based discovery
- Simple leak detection

**Phase 2: Add Canonical Encoding**
- Migrate to PQSF for protocol messages
- Maintain JSON for discovery manifests
- Backward compatible with Phase 1

**Phase 3: Add Time Binding**
- Integrate verifiable time for expiry
- Maintain wall-clock or Unix timestamp compatibility
- Backward compatible with Phase 1–2

**Phase 4: Add Leak Detection**
- Implement exposure monitoring
- Maintain simple mempool polling
- Backward compatible with Phase 1–3

**Phase 5: Add Policy Framework**
- Integrate policy enforcement
- Maintain user-authorized public broadcast
- Backward compatible with Phase 1–4

At each phase, deployments remain interoperable with earlier phases.

---

### I.13 Security Properties Comparison

**Seal-360 Alone**
- Eliminates public mempool exposure during private relay
- Quantum-resistant payload confidentiality (EBT)
- User-authorized public broadcast preserves liveness
- Template hash prevents miner modification
- No policy enforcement

**Seal-360 + PQSF**
- All of the above
- Deterministic encoding
- Algorithm agility

**Seal-360 + PQSF + Verifiable Time**
- All of the above
- Verifiable time progression
- Replay protection

**Seal-360 + PQSF + Verifiable Time + Leak Detection**
- All of the above
- Systematic exposure detection
- Formalized execution-state semantics

**Seal-360 + PQSF + Verifiable Time + Leak Detection + Policy**
- All of the above
- Policy-driven execution control

**Full Stack**
- All of the above
- Multi-phase execution coordination
- Pre-execution authorization

| Property                               | Seal-360 Alone | + PQSF | + PQSF + Time | + PQSF + Time + Leak Detection | Full Stack |
|----------------------------------------|----------------|--------|---------------|--------------------------------|------------|
| Public mempool exposure eliminated     | Yes            | Yes    | Yes           | Yes                            | Yes        |
| Payload confidentiality (EBT)          | Yes            | Yes    | Yes           | Yes                            | Yes        |
| User-authorized public broadcast       | Yes            | Yes    | Yes           | Yes                            | Yes        |
| Transaction byte integrity             | Yes            | Yes    | Yes           | Yes                            | Yes        |
| Deterministic encoding                 | No             | Yes    | Yes           | Yes                            | Yes        |
| Verifiable time anchoring              | No             | No     | Yes           | Yes                            | Yes        |
| Execution-window leak detection        | No             | No     | No            | Yes                            | Yes        |
| Policy-driven execution control        | No             | No     | No            | No                             | Yes        |
| Pre-execution coordination             | No             | No     | No            | No                             | Yes        |

---

### I.14 Implementation Guidance (Informative)

Implementers SHOULD choose integration depth based on deployment needs.

Minimal implementation:
- Seal-360 private relay
- Execution state surfacing on private relay failure
- Template hash binding
- Explicit user or policy authorization before any public broadcast

Advanced implementations MAY layer:
- canonical encoding,
- verifiable time binding,
- exposure detection,
- policy enforcement,
- execution coordination.

No implementation is required to adopt all layers.

Public broadcast, when used, MUST be explicitly authorized and MUST NOT be triggered automatically by any integration.

---

### I.15 Interoperability

All Seal-360 implementations remain interoperable at the protocol level.

- Canonical-encoding wallets can interoperate with JSON-based miners via adapters
- Verifiable-time wallets can interoperate with Unix-timestamp miners
- Leak-aware wallets remain compatible with simple leak detection

Interoperability is preserved through:
- explicit versioning,
- graceful degradation,
- capability negotiation.

---

### I.16 Summary

Seal-360 is a standalone execution-layer protocol designed to operate in Bitcoin environments that have adopted BIP-360.

Stack integration provides optional hardening for organizations deploying comprehensive post-quantum security frameworks.

Use Seal-360 alone for:
- maximum compatibility,
- minimal dependencies,
- straightforward Bitcoin deployment.

Add integrations selectively based on risk tolerance and operational needs.

None of these additions alter Seal-360’s core security properties or deployment model.

---

### I.17 References

Optional integration components:
- PQSF — Canonical encoding and cryptographic suite indirection
- Verifiable time systems
- Exposure detection frameworks
- Policy enforcement kernels
- Execution coordination frameworks

These components are maintained independently and are not required for Seal-360 conformance.

---

## Annex J: Protocol Flow Diagrams

### J.1 Purpose

Protocol flow diagrams are provided to:

- illustrate execution timing and exposure windows,
- clarify miner interaction and verification steps,
- make public broadcast authorization and liveness behavior explicit,
- support security review and audit.


---

### J.2 Actors and Notation

**Actors:**
- **Wallet:** Seal-360–conformant wallet implementation
- **Discovery:** Miner discovery mechanism (Annex H)
- **Miner:** Seal-360–capable miner or mining pool
- **Bitcoin Network:** Public mempool and blockchain
- **User:** Transaction initiator (for notification only)

**Notation:**
- `→` synchronous request/response
- `-->` asynchronous event
- `[STATE]` state transition
- `⏱` timeout
- `✓` success
- `⚠` security-relevant event

---

### J.3 Flow 1: Successful Private Relay (Happy Path)

```

Wallet        Discovery        Miner           Bitcoin Network
│               │              │                    │
│── discover ──>│              │                    │
│<─ offers ─────│              │                    │
│               │              │                    │
│ construct + sign transaction │                    │
│ compute template_hash        │                    │
│ encrypt transaction          │                    │
│──────── Encrypted Submission ────────────────→   │
│               │              │ decrypt + verify   │
│               │              │ include in block   │
│               │              │───────────────→   │
│               │              │                    │
│<──────── confirmation observed on-chain ─────────│
│ [SUCCESS] ✓

```

**Properties:**
- No public mempool exposure
- Signatures visible only after confirmation
- Zero execution-window exposure

---

### J.4 Flow 2: Timeout and User Authorization

```

Wallet               User               Miner              Bitcoin Network
│                     │                  │                     │
│ Encrypted Submission│                  │                     │
│──────────────────────────────────────→ │                     │
│                     │                  │ (no inclusion)      │
│                     │                  │                     │
│ ⏱ timeout exceeded  │                  │                     │
│                     │                  │                     │
│ surface execution   │                  │                     │
│ state               │                  │                     │
│───────────────────→ │                  │                     │
│                     │                  │                     │
│ [User chooses RETRY]│                  │                     │
│<────────────────────│                  │                     │
│                     │                  │                     │
│ retry private relay │                  │                     │
│──────────────────────────────────────→ │                     │

OR

│ [User authorizes    │                  │                     │
│  public broadcast] │                  │                     │
│<────────────────────│                  │                     │
│                     │                  │                     │
│ show quantum risk   │                  │                     │
│ warning             │                  │                     │
│───────────────────→ │                  │                     │
│<─ user confirms ────│                  │                     │
│                     │                  │                     │
│ broadcast publicly  │──────────────────────────────────────→ │
│                     │                  │                     │
│<──────── confirmation observed ────────────────────────────────│
│ [SUCCESS with public broadcast] ⚠

```

**Properties:**
- No automatic public broadcast occurs.
- Execution state is surfaced for explicit user or policy decision.
- Public broadcast requires an Explicit Public Broadcast Authorization Event.
- Quantum exposure occurs only after explicit authorization.
- Authorized public broadcast may introduce additional delay or correlation relative to immediate public broadcast.

---

### J.5 Flow 3: Premature Leak Detection

```

Wallet            Miner           Bitcoin Network
│                 │                    │
│ Encrypted Submission                │
│───────────────→ │                    │
│                 │ leaks transaction  │
│                 │───────────────→   │
│                 │                    │
│ detect mempool appearance ⚠          │
│ notify user                          │
│ optional RBF / wait                 │
│<──────── confirmation observed ─────│
│ [SUCCESS with leak] ⚠

```

**Properties:**
- Exposure detected
- User informed
- Miner reputation may be updated
- Transaction still confirms

---

### J.6 Exposure Window Comparison

| Scenario                               | Public Mempool Exposure        | Execution Window |
|----------------------------------------|-------------------------------|------------------|
| Standard Bitcoin                       | Always                        | ~1 block        |
| Seal-360 (private relay succeeds)      | None prior to confirmation    | 0               |
| Seal-360 (public broadcast authorized) | After explicit authorization  | ~1 block        |
| Seal-360 (premature leak)              | From leak onward              | Variable        |

Seal-360 does not introduce automatic public mempool exposure.

While private relay succeeds, public mempool exposure is avoided during the
execution window.

If public broadcast is explicitly authorized, execution behavior becomes
equivalent to standard Bitcoin for that transaction, including public mempool
visibility during the execution window.

---

### J.7 State Machine Summary

```

DISCOVERED
↓
SIGNED
↓
SUBMITTED
↓
PENDING
↓
[ INCLUDED ] → SUCCESS
[ TIMEOUT ]  → SURFACE_DECISION
[ LEAKED ]   → SUCCESS (with exposure)

```

All non-success paths surface execution state and require explicit user or policy authorization before any public broadcast. No automatic transition to public broadcast exists.

---

### J.8 Liveness and Safety Guarantees

- At least one execution option always remains available through explicit user or policy authorization.
- No silent failure or deadlock is permitted.
- Public broadcast is never triggered automatically by the protocol.
- User-authorized public broadcast preserves Bitcoin’s censorship resistance.
- Quantum exposure occurs only after explicit authorization.
- All failure modes surface execution state and defer action to the user or policy engine.

---

### J.9 Error Handling Paths

**Template hash mismatch:**
- Miner rejects submission
- Wallet surfaces execution state and allows retry or user-authorized public broadcast

**Transport failure:**
- Wallet retries submission
- On repeated failure, execution state is surfaced for explicit authorization

**Discovery failure:**
- Wallet skips private relay
- Public broadcast authorized explicitly by user or policy

All errors result in either retry or user-authorized public broadcast. No transaction may be silently dropped.

---

### J.10 Multi-Miner Routing (Privacy Enhancement)

For wallets implementing enhanced privacy or availability, miner discovery MAY be performed in parallel:

```

Wallet
├── Miner A
├── Miner B
└── Miner C
↓
Select based on fee, latency, reputation
↓
Submit to chosen miner

```

**Benefits:**
- Reduced correlation
- Improved resilience to refusal
- Reduced dependence on a single miner

This behavior is optional and does not alter core protocol guarantees.

---

### J.11 Integration with BIP-360

Seal-360 is designed to complement BIP-360.

**Combined execution path:**
```

Receive to BIP-360 output
→ Spend via Seal-360 private relay
→ Send to BIP-360 output

```

**Result:**
- Output-layer exposure removed (BIP-360)
- Execution-layer exposure removed (Seal-360)
- No additional consensus changes beyond those required by BIP-360

Without Seal-360, BIP-360 alone leaves the execution window exposed.

---

### J.12 Summary

These diagrams demonstrate that Seal-360:

- eliminates execution-window exposure when private relay succeeds,
- preserves Bitcoin’s liveness and censorship resistance,
- surfaces failures transparently and allows user-controlled degradation to standard Bitcoin when public broadcast is explicitly authorized,
- introduces no new consensus, custody, or authorization assumptions.

---

## Annex K: Optional PQHD Keymail Transport (Informative)

This annex describes an optional transport mechanism for Seal-360 Encrypted Submission using PQHD Keymail or equivalent cryptographically addressed mailbox systems.

Nothing in this annex is required for Seal-360 conformance.

### K.1 Purpose

PQHD Keymail transport provides an alternative delivery mechanism for Encrypted Submission that does not rely on classical web PKI or synchronous HTTPS request/response semantics.

This annex exists for deployments that require:

* cryptographic addressing instead of certificate-based endpoint identity,
* reduced reliance on classical public key infrastructure,
* asynchronous or store-and-forward delivery models,
* alignment with post-quantum identity or custody architectures.

### K.2 Transport Semantics

When PQHD Keymail transport is used:

* the miner advertises a cryptographic Keymail address bound to its identity,
* the wallet encrypts the Encrypted Submission payload exactly as specified in Section 9.5,
* AEAD associated data construction remains unchanged (Section 9.5.3),
* delivery occurs via Keymail message delivery rather than HTTPS request/response.

PQHD Keymail transport affects delivery only and MUST NOT alter:

* transaction construction or signing,
* `template_hash` computation,
* authorization semantics,
* fallback or failure behavior.

### K.3 Authentication Model

Under PQHD Keymail transport:

* miner authentication is derived from cryptographic address binding,
* endpoint authenticity does not depend on TLS certificate authorities,
* replay and expiration semantics remain enforced via AEAD associated data.

Compromise or misrouting at the transport layer MAY result in denial-of-service or delivery failure, but MUST NOT result in automatic public broadcast or loss of signing authority.

### K.4 Conformance and Interoperability

Use of PQHD Keymail transport is OPTIONAL.

Wallets and miners claiming Seal-360 conformance MUST continue to support at least one authenticated baseline transport mechanism as defined in Section 9.6.

PQHD Keymail transport MAY be supported in parallel with HTTPS/TLS or other authenticated transports.

---

### K.5 Related Specifications (Informative)

Seal-360 is designed to operate independently.  
The following specifications are referenced for alignment or optional extension only. None are required for Seal-360 conformance.

#### Directly Relevant

**PQHD — Post-Quantum Hierarchical Deterministic Wallet**  
https://github.com/rosieRRRRR/pqhd

PQHD defines post-quantum identity, key derivation, and cryptographically addressed messaging (Keymail).  
PQHD Keymail MAY be used as an optional transport for Encrypted Submission as described in this annex.

---

**Bitcoin Pre-Contracts (BPC)**  
https://github.com/rosieRRRRR/BPC

BPC introduced execution-first design principles that influenced Seal-360’s separation of transaction construction from execution delivery.  
Seal-360 does not require BPC and does not implement preconditions or refusal logic.

---

#### Conceptually Aligned (Not Integrated)

The following specifications address broader post-quantum or execution-control concerns.  
They are not integrated with Seal-360 v1.0.0 and impose no requirements.

* **PQSF — Post-Quantum Serialization Format**  
  https://github.com/rosieRRRRR/pqsf  
  Canonical encoding and cryptographic agility.

* **PQSEC — Post-Quantum Security Policy Framework**  
  https://github.com/rosieRRRRR/PQSEC  
  Policy and enforcement concepts layered above protocols.

* **PQEH — Post-Quantum Execution Harness**  
  https://github.com/rosieRRRRR/PQEH  
  Execution coordination and multi-phase execution models.

* **Epoch Clock**  
  https://github.com/rosieRRRRR/epoch-clock  
  Verifiable time anchoring and replay-bound semantics.

Other research repositories and experimental systems are intentionally omitted to preserve scope clarity.

---

## Annex L: Minimum Viable Implementation (Informative)

This annex describes the minimum functionality required to demonstrate a
working Seal-360 execution path. It is informative only.

Nothing in this annex defines recommended defaults, preferred architectures,
or production best practices. It exists solely to clarify the smallest
implementable subset of the specification.

Seal-360 conformance is defined exclusively by Sections 12 and 14.

---

### L.1 Minimum Viable Seal-360 Wallet

A minimal Seal-360 wallet implementation MUST be capable of:

- constructing a standard, broadcast-valid Bitcoin transaction,
- computing `template_hash` exactly as specified in Section 9.3,
- encrypting the signed transaction payload using ML-KEM-768 and AEAD,
- performing Encrypted Direct-to-Miner Submission to a single authenticated miner,
- surfacing execution state on private relay failure,
- requiring explicit user authorization before any public broadcast.

A minimum implementation does NOT require:

- multi-miner routing,
- miner reputation tracking,
- policy frameworks,
- canonical encoding beyond Section 9.5.3,
- exposure detection,
- execution coordination.

---

### L.2 Minimum Viable Seal-360 Miner

A minimal Seal-360 miner implementation MUST be capable of:

- publishing an ML-KEM-768 public encryption key,
- accepting encrypted submissions over an authenticated transport,
- reconstructing canonical AAD and performing AEAD decryption,
- recomputing and verifying `template_hash`,
- rejecting invalid, replayed, or expired submissions,
- validating transactions under standard Bitcoin consensus rules,
- including accepted transactions in block templates.

A minimum implementation does NOT require:

- coordination with other miners,
- any commitment to inclusion,
- any change to Bitcoin consensus behavior.

---

### L.3 Interpretation

A single-wallet, single-miner deployment that satisfies the requirements above
is sufficient to demonstrate execution-window exposure reduction for
quantum-sensitive or high-value transactions.

This annex does not imply that such a deployment is optimal, recommended, or
appropriate for all environments.

Seal-360 remains correct, interoperable, and conformant regardless of whether
deployments exceed this minimum.

---

## Annex M: Reference Code, Visual Summary, and Naming (Informative)

This annex is informative only. It provides Rust-focused reference snippets and presentation guidance to accelerate independent implementations. Nothing in this annex modifies Seal-360 conformance requirements.

This annex is not a required implementation. It does not define transport, discovery, fee policy, or user authorization behavior. Normative requirements remain in Sections 9–12.

---

### M.1 Visual Summary (Header Diagram)

```

WITHOUT SEAL-360:
Sign → Broadcast → [PUBLIC MEMPOOL] → Confirm
↑ signatures visible during execution window

WITH SEAL-360:
Sign → Encrypt (ML-KEM + AEAD) → Direct to Miner → Confirm
↑ no public exposure prior to confirmation

```

---

### M.2 Reference: `template_hash` Computation (Section 9.3)

Reference computation:

```

template_hash = SHA256(raw_tx_bytes)

```

Rust reference:

```rust
use sha2::{Digest, Sha256};

pub fn template_hash(raw_tx_bytes: &[u8]) -> [u8; 32] {
    let mut hasher = Sha256::new();
    hasher.update(raw_tx_bytes);
    let out = hasher.finalize();
    let mut digest = [0u8; 32];
    digest.copy_from_slice(&out);
    digest
}
```

---

### M.3 Reference: Canonical AAD Encoding (RFC 8785 JCS) (Section 9.5.3)

Seal-360 AAD is the canonical JSON encoding (RFC 8785 JCS) of the following fields:

* `template_hash` (32-byte hex string, lowercase)
* `submission_id` (16-byte hex string, lowercase)
* `miner_id` (string)
* `key_id` (string)
* `expires_at` (string, ISO 8601)

Reference canonical JSON string (example):

```json
{"expires_at":"2026-02-15T00:00:00Z","key_id":"2026-01-key-01","miner_id":"example-pool-01","submission_id":"00112233445566778899aabbccddeeff","template_hash":"0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef"}
```

Rust reference (string-only JCS-safe canonicalization):

```rust
use hex::ToHex;

/// Produces canonical JSON bytes for Seal-360 AAD when all values are strings.
/// Keys are emitted in lexicographic order and values are JSON-escaped.
///
/// This approach is RFC 8785-compatible for string-only objects and avoids
/// numeric canonicalization pitfalls.
pub fn aad_canonical_bytes(
    template_hash_32: [u8; 32],
    submission_id_16: [u8; 16],
    miner_id: &str,
    key_id: &str,
    expires_at: &str,
) -> Vec<u8> {
    let th_hex = template_hash_32.encode_hex::<String>();
    let sid_hex = submission_id_16.encode_hex::<String>();

    let expires_at_js = serde_json::to_string(expires_at).expect("json string");
    let key_id_js = serde_json::to_string(key_id).expect("json string");
    let miner_id_js = serde_json::to_string(miner_id).expect("json string");
    let submission_id_js = serde_json::to_string(&sid_hex).expect("json string");
    let template_hash_js = serde_json::to_string(&th_hex).expect("json string");

    let s = format!(
        "{{\"expires_at\":{},\"key_id\":{},\"miner_id\":{},\"submission_id\":{},\"template_hash\":{}}}",
        expires_at_js, key_id_js, miner_id_js, submission_id_js, template_hash_js
    );
    s.into_bytes()
}
```

---

### M.4 Reference: Encryption and Verification Flow (ML-KEM + AEAD)

This section illustrates a reference structure for the encryption and verification flow. It is not a mandated wire format.

Reference envelope fields:

* `kem_ct`: ML-KEM-768 ciphertext (encapsulation output)
* `nonce`: 24 bytes (XChaCha20-Poly1305 nonce)
* `ciphertext`: AEAD ciphertext (encrypted transaction bytes)
* `miner_id`: selected miner identity
* `key_id`: selected miner key identifier
* `expires_at`: offer expiry timestamp
* `submission_id`: unique per submission attempt
* `template_hash`: SHA256(raw_tx_bytes)

Rust-like reference pseudocode (encryption):

```rust
// PSEUDOCODE: choose a concrete ML-KEM-768 implementation and an AEAD crate
// supporting XChaCha20-Poly1305.

fn seal360_encrypt_submission(
    miner_mlkem_pubkey: &[u8],
    raw_tx_bytes: &[u8],
    miner_id: &str,
    key_id: &str,
    expires_at: &str,
) -> EncryptedSubmission {
    // 1) Compute template_hash
    let template_hash32 = template_hash(raw_tx_bytes);

    // 2) Generate submission_id (16 bytes random)
    let submission_id16 = random_16_bytes();

    // 3) ML-KEM-768 encapsulate (KEM)
    let (kem_ct, shared_secret) = mlkem768_encapsulate(miner_mlkem_pubkey);

    // 4) Derive AEAD key material (HKDF over shared_secret)
    let aead_key = hkdf_expand_32(&shared_secret, b"seal360/aead_key");

    // 5) Construct canonical AAD bytes (RFC 8785 JCS)
    let aad = aad_canonical_bytes(template_hash32, submission_id16, miner_id, key_id, expires_at);

    // 6) AEAD encrypt raw_tx_bytes using XChaCha20-Poly1305 with AAD
    let nonce24 = random_24_bytes();
    let ciphertext = xchacha20poly1305_encrypt(&aead_key, &nonce24, &aad, raw_tx_bytes);

    EncryptedSubmission {
        kem_ct,
        nonce: nonce24,
        ciphertext,
        template_hash: template_hash32,
        submission_id: submission_id16,
        miner_id: miner_id.to_string(),
        key_id: key_id.to_string(),
        expires_at: expires_at.to_string(),
    }
}
```

Miner-side reference verification steps:

* Reconstruct canonical AAD bytes identically using the received fields.
* AEAD-decrypt under that AAD.
* Recompute `template_hash` over the decrypted transaction bytes and compare.
* Enforce replay cache on (`miner_id`, `submission_id`) until at least `expires_at`.

---

### M.5 Reference: Miner Replay Cache Semantics (Section 12.2)

Reference replay protection model:

* Key: (`miner_id`, `submission_id`)
* Lifetime: until at least `expires_at`
* Behavior: reject any repeated key within lifetime

Rust reference (in-memory, illustrative):

```rust
use std::collections::HashMap;
use chrono::{DateTime, Utc};
use hex::ToHex;

#[derive(Default)]
pub struct ReplayCache {
    // key -> expires_at
    seen: HashMap<String, DateTime<Utc>>,
}

impl ReplayCache {
    pub fn check_and_mark(
        &mut self,
        miner_id: &str,
        submission_id: [u8; 16],
        expires_at: &str,
        now: DateTime<Utc>,
    ) -> Result<(), String> {
        let exp = DateTime::parse_from_rfc3339(expires_at)
            .map_err(|_| "expires_at is not valid RFC3339/ISO8601".to_string())?
            .with_timezone(&Utc);

        if now > exp {
            return Err("expired offer".to_string());
        }

        let key = format!("{}:{}", miner_id, submission_id.encode_hex::<String>());

        if let Some(prev_exp) = self.seen.get(&key) {
            if *prev_exp >= now {
                return Err("replay detected".to_string());
            }
        }

        self.seen.insert(key, exp);
        Ok(())
    }

    pub fn purge_expired(&mut self, now: DateTime<Utc>) {
        self.seen.retain(|_, exp| *exp >= now);
    }
}
```

Production implementations MUST persist replay state across restarts for at
least the duration of the offer validity window.

---

### M.6 Naming Consistency (Editorial Guidance)

Use the following terms consistently in discussion, documentation, and implementations:

* **Seal-360**: protocol/spec name.
* **Encrypted Direct-to-Miner Submission** (or **Encrypted Submission**): the send-to-miner action (Section 1.1).
* **Private relay**: the overall execution mode (discovery → submit → include), excluding any automatic public broadcast.
* **Public broadcast**: submission to the public Bitcoin network and mempool, permitted only via an Explicit Public Broadcast Authorization Event.

Avoid introducing new synonyms (for example “PR”, “private broadcast”, “securecast”, “quancast”) in public discussion or implementation naming.

---

## Annex N: Relationship to Bitcoin Pre-Contracts (Informative)

Seal-360 is conceptually aligned with the Bitcoin Pre-Contracts design philosophy,
specifically the separation of execution delivery from transaction construction.

> **A broadcast-valid transaction MUST NOT be publicly visible prior to intended execution.**

Seal-360 applies this invariant without introducing:

* precondition evaluation,
* refusal logic,
* authority delegation,
* policy enforcement.

Transactions are constructed and signed normally.
Public broadcast is deferred, not prevented.
Execution is attempted via private relay, with public broadcast available only
through explicit user or policy authorization.

### Censorship Resistance

Seal-360 does not remove the ability to submit valid transactions to Bitcoin’s
public network; it removes automatic submission.

Bitcoin’s censorship resistance is preserved through the following properties:

* User-authorized public broadcast is always available.
* No protocol-level prohibition exists on public submission.
* No miner coordination, permission, or cooperation is required for public broadcast.
* Standard Bitcoin relay and mempool propagation remain unchanged and accessible.

Any implementation that removes or disables user-initiated public broadcast is
non-conformant.

---

## 15. Normative References

- **FIPS 203:** Module-Lattice-Based Key-Encapsulation Mechanism (ML-KEM)
- **RFC 8785:** JSON Canonicalization Scheme (JCS)
- **BIP-125:** Opt-in Full Replace-by-Fee Signaling
- **BIP-141:** Segregated Witness (Consensus layer)

---

## 16. Informative References

- **BIP-360:** Pay-to-Tapscript-Hash specification (bip360.org)
- **Bitcoin Pre-Contracts (BPC):** Full specification available separately

---

## Acknowledgements

This work builds on prior research and discussion around Bitcoin execution security, post-quantum cryptography, and mempool exposure risks.

Thanks to:

* the **BIP-360 authors and reviewers** for advancing output-layer quantum hardening,
* Bitcoin Core developers and researchers for sustained work on mempool behavior and execution semantics,
* post-quantum cryptography contributors whose work informs practical transport-layer defenses,
* reviewers and operators who provided critical feedback on miner interaction, public broadcast authorization behavior, and threat boundaries.


Any errors or omissions remain the responsibility of the author.
