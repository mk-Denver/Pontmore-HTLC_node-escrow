# Pontmore HTLC Escrow — Implementation Guide

**Building the Pontmore-compatible Lightning hold-invoice escrow described by [`README.md`](../README.md).**

This guide describes the implementation of this repository's node-controlled **Lightning hold-invoice escrow** for Pontmore peer-to-peer Bitcoin/fiat swaps. Read [`README.md`](../README.md) first for the project overview, custody model, protocol stack, architecture, and limitations. This document expands the implementation details and must remain consistent with that README.

This project is **not** an agent-controlled hold-invoice construction and must not be documented or advertised as a trustless escrow. The escrow operator controls the Lightning node and the settlement capability. Neither trading participant receives the escrow preimage or direct access to the node's settlement credentials.

The implementation is experimental and depends on proving that the selected LDK version supports the required incoming hold-invoice lifecycle.

> **Protocol source of truth:** The canonical [Pontmore protocol repository](https://github.com/pontmore/protocol) and the exact specification versions pinned by a coordination are authoritative. This repository's [`README.md`](../README.md) describes the intended implementation and custody boundary; it does not replace the Pontmore specifications.

---

## 1. Implementation identity and custody model

The escrow descriptor for this implementation MUST advertise:

```text
escrow_type: lightning_hold_invoice
network: lightning
```

Do not use `custodial_escrow` in the descriptor, service schema, implementation guide, or conformance claims for this project. The relevant Pontmore PIP-01 subtype is `lightning_hold_invoice`.

The term “node-controlled” describes the trust boundary: the escrow operator controls the node and the hold-invoice settlement capability. It does not mean that the customer or Agent controls the invoice preimage, and it does not provide cryptographic arbitration by itself.

The intended lifecycle is:

1. The escrow service creates or arranges a Lightning hold invoice.
2. The service securely retains the settlement capability required by the selected construction.
3. The customer pays the funding invoice.
4. The LDK node reports the actual incoming payment and HTLC state.
5. A valid Pontmore authorization permits settlement or refund according to the pinned profile and service schema.
6. The daemon invokes the supported Lightning operation.
7. The daemon reconciles the external result before publishing a final coordination action.

A service-controlled preimage is a secret. It MUST NOT be published to Nostr, sent to trading participants, or included in public evidence.

For the project-level description of this boundary, see [`README.md`, section 2 — Custody model](../README.md#2-custody-model).

---

## 2. Architecture and responsibilities

```text
Customer                         Agent
   │                               │
   │ Pays hold invoice             │ Provides payout invoice
   ▼                               ▼
┌───────────────────────────────────────────┐
│          Pontmore Escrow Daemon           │
│                                           │
│  Authorization · State validation         │
│  Durable execution · Recovery             │
│  Payment reconciliation                   │
└─────────────────────┬─────────────────────┘
                      │
                      ▼
┌───────────────────────────────────────────┐
│             LDK-based Node                │
│                                           │
│  Hold invoice · HTLC lifecycle             │
│  Channel and chain monitoring             │
└─────────────────────┬─────────────────────┘
                      │
                      ▼
               Lightning Network

Nostr coordination:
  PIP-00 Agent definition
       → PIP-01 lightning_hold_invoice descriptor
       → PIP-02 v2 coordination root and action chain
       → pontmore/swap@1 profile
       → versioned hold-invoice service schema
```

| Component | Responsibility |
| --- | --- |
| LDK node | Creates or supports the hold-invoice construction, observes payment/HTLC state, and performs supported claim/failure operations. |
| Escrow daemon | Enforces service authorization, persists execution intent, invokes permitted Lightning operations, and reconciles outcomes. |
| Authorization engine | Validates signatures, role bindings, profile state, deadlines, commitments, predecessor linkage, and economic authorization. |
| Coordination validator | Validates PIP-02 v2 roots and linked actions and detects forks or invalid chains. |
| Nostr publisher | Signs and publishes permitted public coordination events. |
| Private messaging | Transports invoices, payout instructions, and sensitive evidence. |
| Persistent storage | Stores swap state, payment identifiers, authorization references, idempotency keys, and recovery checkpoints. |
| Recovery worker | Reconciles actual Lightning state after crashes, expiry, failures, and ambiguous responses. |
| Resolver | Publishes a permitted `core/resolve_dispute` effect when authorized; it receives no Lightning credentials. |

The daemon MUST NOT treat a published Nostr event as proof that a Lightning payment succeeded.

The broader repository architecture is documented in [`README.md`, section 3 — Architecture](../README.md#3-architecture).

---

## 3. Pontmore protocol dependencies

| Resource | Use in this implementation |
| --- | --- |
| [PIP-00 — Agent Definition](https://github.com/pontmore/protocol/blob/main/PIP-00-agent-definition.md) | Publishes and discovers Agent capabilities and references to escrow descriptors. |
| [PIP-01 — Escrow Descriptor](https://github.com/pontmore/protocol/blob/main/PIP-01-escrow-descriptor.md) | Advertises `lightning_hold_invoice`, the `lightning` network, descriptor expiry, and the versioned service schema. |
| [PIP-02 — Coordination Event Chains](https://github.com/pontmore/protocol/blob/main/PIP-02-coordination-event-chains.md), version 2 | Defines kind `7300` roots, kind `7301` actions, authority binding, linkage, authorization invariants, expiry, forks, disputes, and resolution effects. |
| [`pontmore/swap@1`](https://github.com/pontmore/protocol/blob/main/profiles/swap-v1.md) | Experimental swap profile defining terms, roles, `swap/fiat_sent`, `swap/fiat_confirmed`, deadlines, dispute classes, and authorization gates. This is a profile, not a PIP. |
| Referenced hold-invoice service schema | Defines the concrete API, authentication, funding state, hold-invoice behavior, evidence handling, resolver binding, timeout/recovery, reconciliation, fees, and quote rules. |

There is no PIP-03 in the current Pontmore protocol. PIP-02 supplies the shared dispute kernel and `core/resolve_dispute`; the pinned profile and accepted service schema supply domain-specific dispute and service behavior.

The implementation MUST pin exact PIP, profile, descriptor-event, commitment, quote, and service-schema versions and MUST reject unsupported versions.

### PIP-02 v2 event model

PIP-02 v2 defines only:

* Kind `7300`: immutable coordination root.
* Kind `7301`: immutable coordination action.

Evidence, disputes, and notes do not have separate canonical event kinds. Evidence is referenced by actions, disputes are actions, and operational notes remain private or application-local.

Kernel actions include:

```text
core/accept
core/decline
core/secure
core/authorize_settlement
core/settle
core/authorize_refund
core/refund
core/cancel
core/expire
core/open_dispute
core/resolve_dispute
```

`pontmore/swap@1` defines `swap/fiat_sent` and `swap/fiat_confirmed` as profile actions. They are not separate event kinds.

`core/resolve_dispute.data` requires a `policy` and an `effect` of `resume`, `authorize_settlement`, `authorize_refund`, or `cancel`. A resolution effect does not move funds; the bound escrow authority must still execute and publish `core/settle` or `core/refund`.

---

## 4. Roles and authorization

The root binds namespaced roles, not informal UI aliases:

| Role | Responsibility |
| --- | --- |
| `swap/customer` | Customer participant. Its economic role depends on the swap direction. |
| `swap/agent` | Agent participant. Its economic role depends on the swap direction. |
| `core/escrow` | Bound escrow authority that publishes `core/secure`, `core/settle`, and `core/refund`. |
| `core/resolver` | Bound resolver authority that publishes `core/resolve_dispute` when disputes are enabled. |

For `pontmore/swap@1`:

* `fiat_to_btc`: the customer sends fiat and the Agent provides Bitcoin.
* `btc_to_fiat`: the Agent sends fiat and the customer provides Bitcoin.

The resolver is not the Lightning executor. The resolver's key MUST be separate from the node and settlement credentials.

Under the swap profile:

* The fiat receiver publishes `core/authorize_settlement` after valid `swap/fiat_confirmed`.
* The Bitcoin provider may publish `core/authorize_refund` after `fiat_pay_by` elapses without valid `swap/fiat_sent`.
* The bound escrow authority publishes `core/settle` or `core/refund` after executing and reconciling the corresponding Lightning operation.
* A dispute freezes ordinary progression until a valid `core/resolve_dispute` effect is recorded.

---

## 5. PIP-01 lightning hold-invoice descriptor

Before accepting swaps, publish a kind `30361` addressable descriptor whose `escrow_type` is `lightning_hold_invoice`.

Illustrative content:

```json
{
  "version": 1,
  "escrow_type": "lightning_hold_invoice",
  "networks": ["lightning"],
  "service": {
    "schema": {
      "type": "openapi",
      "url": "https://escrow.example.com/schemas/ldk-hold-invoice-v1.json"
    }
  },
  "expires_at": 1780000000
}
```

This is illustrative, not a deployed descriptor. The actual event MUST include the required `d` tag and discovery tags, and the schema URL MUST identify the actual versioned service schema.

Descriptor validation MUST include:

1. Addressable-event structure and stable `d` tag.
2. `escrow_type: lightning_hold_invoice`.
3. `networks` containing `lightning`.
4. A versioned OpenAPI or AsyncAPI schema.
5. An absolute HTTPS schema URL.
6. Fetch-safety checks for private destinations, redirects, content type, and response size.
7. Exact binding of the accepted descriptor event ID and descriptor coordinate in every coordination.

The descriptor MUST NOT contain preimages, raw invoices, payout instructions, credentials, private evidence, or internal reconciliation records.

The service schema owns implementation-specific behavior, including hold-invoice creation, funding status, partial funding, release, refund, cancellation, resolver binding, evidence submission, timeout/recovery, reconciliation, fees, and quote formats.

See [`README.md`, section 4 — PIP-01 escrow descriptor](../README.md#4-pip-01-escrow-descriptor).

---

## 6. Coordination root and action chain

A swap root is a kind `7300` PIP-02 v2 event that pins `pontmore/swap@1` and binds:

* exact swap terms and direction;
* `swap/customer` and `swap/agent` identities;
* exactly one `core/escrow` authority;
* a `core/resolver` authority when disputes are enabled;
* the exact accepted descriptor event and its coordinate;
* supported protocol/profile versions;
* commitments, quotes, and deadlines where applicable.

The accepting participant validates the root, Agent definition, descriptor, service schema, commitments, authority bindings, and profile before publishing `core/accept`.

Every action is a kind `7301` event containing a root reference and an immediate predecessor reference. Relay order, event arrival order, timestamps, and event-ID ordering do not establish chain order.

Reject unsupported versions, invalid signatures, invalid commitments, unauthorized signers, invalid predecessors, replayed actions, and actions inconsistent with the derived state.

If two valid actions reference the same predecessor, retain both as evidence of a fork and freeze further economic action. Never select a winner by relay order.

---

## 7. Lightning hold-invoice lifecycle

### 7.1 Feasibility gate

Before building the production service, prove on Bitcoin regtest that the exact pinned LDK implementation can:

* create or support the required incoming hold invoice;
* keep the incoming payment pending until application authorization;
* retain the settlement capability securely;
* claim the payment only after valid authorization;
* fail or cancel the held payment when permitted;
* recover payment state after restart;
* handle HTLC expiry and CLTV constraints;
* reconcile ambiguous operation results without duplicate execution.

A normal invoice API or generic inbound-payment callback is not sufficient evidence of hold-invoice support.

### 7.2 Funding

1. Generate the preimage using a cryptographically secure random source when required by the construction.
2. Compute the payment hash.
3. Create the hold invoice through the supported LDK/backend API.
4. Persist the invoice reference, hash, amount, expiry, and encrypted preimage where applicable.
5. Deliver the invoice privately to the customer.
6. Observe payment and HTLC state from the node.
7. Verify amount, payment hash, completion condition, and any multipart policy.
8. Publish `core/secure` only after the profile/service-defined Bitcoin security condition is satisfied.

Invoice creation, delivery, or partial payment observation does not prove that the escrow is secured.

### 7.3 Claim and failure

The daemon MUST persist an idempotent execution intent before invoking a claim or failure operation. It MUST reconcile the actual node state before retrying after a timeout or process failure.

The exact API is backend-specific. Do not copy LND commands into an LDK implementation and assume equivalent semantics.

---

## 8. Swap execution

### 8.1 Normal release

For `pontmore/swap@1`:

1. The fiat sender performs the agreed fiat-side obligation.
2. The fiat sender publishes `swap/fiat_sent` before `fiat_pay_by`.
3. The fiat receiver confirms receipt and publishes `swap/fiat_confirmed` before `fiat_confirm_by`.
4. The fiat receiver publishes `core/authorize_settlement`.
5. The daemon validates the action chain, signer, profile state, deadline, and service authorization.
6. The daemon persists an idempotent execution intent.
7. The escrow authority claims/settles the held Lightning payment through the supported API.
8. The daemon reconciles the incoming result.
9. The daemon attempts the Agent payout as a separate operation.
10. The escrow authority publishes `core/settle` only after the final outcome satisfies the accepted service schema.

A successful incoming claim does not prove that the Agent received the payout. The daemon must expose and persist an independent payout-pending state where necessary.

### 8.2 Refund

The Bitcoin provider may authorize a refund after the profile's no-payment condition is satisfied. A valid dispute resolution may also derive `authorize_refund`.

A timeout, HTLC expiry, node restart, or failed RPC request is not by itself a refund authorization. If the Lightning payment has already failed or expired, record the verified external fact and follow the valid profile/service recovery path. Do not fabricate a refund action.

The escrow authority publishes `core/refund` only after a valid refund authorization and verified execution outcome.

### 8.3 Cancellation and expiry

`core/cancel` is distinct from refund and is used only when permitted before a final economic outcome. `core/expire` records a profile-defined expiry/recovery condition. Neither action should be substituted for another merely because a timeout occurred.

---

## 9. Disputes and evidence

PIP-02 provides the dispute kernel. The pinned profile defines domain dispute classes and permitted effects, and the accepted service schema defines evidence transport, evaluation policy, resolver binding, and service operations.

### Opening a dispute

An authorized participant publishes `core/open_dispute`. The daemon validates the participant, root, predecessor, derived state, profile class, deadline, and service requirements.

A valid dispute freezes ordinary profile progress, settlement authorization, settlement, and refund until a valid `core/resolve_dispute` effect is recorded.

### Evidence

Evidence travels through the private service/profile channel and may include fiat receipts, delivery confirmations, signed messages, attestations, timestamps, or opaque references. Evidence references are not automatically proof of the underlying event.

Public actions MUST contain only the minimum data needed for validation. Never publish raw invoices, payment credentials, account details, preimages, or raw evidence documents.

### Resolution

The resolver publishes a linked `core/resolve_dispute` action containing:

* `policy`: the policy identifier bound by the profile, root, or accepted service;
* `effect`: `resume`, `authorize_settlement`, `authorize_refund`, or `cancel`.

The daemon verifies resolver authority, signature, root and predecessor linkage, disputed state, policy binding, permitted effect, deadlines, and replay protection.

A resolution effect is an authorization/state transition, not a Lightning payment. If it authorizes settlement or refund, the escrow authority must still execute and publish the corresponding final action.

---

## 10. Timeouts, HTLC expiry, and recovery

Keep these clocks and conditions separate:

* descriptor selection expiry;
* root acceptance expiry;
* funding invoice validity;
* actual accepted HTLC expiry and CLTV constraints;
* `fiat_pay_by`;
* `fiat_confirm_by`;
* dispute and service-resolution deadlines;
* execution and recovery safety margins.

Wall-clock timestamps and block-height/CLTV values are different units and MUST NOT be compared directly.

The service schema must ensure that any permitted resolution window fits within the actual hold-invoice and HTLC constraints. A dispute cannot safely remain open beyond the point at which the held payment must fail or be recovered.

Recovery invariants:

* External Lightning state does not itself authorize a Pontmore action.
* HTLC expiry or failure does not independently authorize `core/refund`.
* Absence of `swap/fiat_confirmed` is not proof that fiat was not received.
* A timeout does not select a default winner unless the profile/service explicitly defines the authorization and recovery path.
* A lost RPC response is not proof of failure.
* A restart does not authorize replaying an economic operation.
* Unknown payment state requires reconciliation before final publication.
* Forks and invalid chains freeze unsafe economic progression.

For the repository's operational limitations, see [`README.md`, sections 9 and 12](../README.md#9-timeouts-htlc-expiry-and-recovery).

---

## 11. Persistence and idempotency

Persist at least:

* coordination root ID and exact descriptor event ID/coordinate;
* customer and Agent identities;
* escrow and resolver authority bindings;
* pinned PIP-02/profile versions;
* terms, commitments, quotes, and deadlines;
* funding invoice reference and payment hash;
* encrypted preimage where required;
* actual HTLC expiry information;
* current validated action-chain tip and derived state;
* authorization event and effect;
* incoming payment state;
* outgoing payout invoice/payment state;
* durable execution intent and idempotency key;
* reconciliation checkpoints and final verified outcome.

Use transactions or compare-and-swap controls to prevent competing workers from executing incompatible operations. The database is an execution journal; it does not override the validated public chain or actual Lightning state.

---

## 12. Service schema and API boundary

The PIP-01 descriptor MUST reference the versioned OpenAPI or AsyncAPI schema for this hold-invoice service. The schema must define:

* transport, authentication, and authorization;
* escrow instance creation and participant binding;
* private hold-invoice delivery and funding verification;
* claim, failure, refund, cancellation, and partial outcomes;
* payout invoice submission and replacement rules;
* evidence submission and resolver binding;
* supported dispute-resolution effects;
* timeout, HTLC-expiry, and recovery behavior;
* idempotency, errors, retries, and reconciliation;
* fees and exact quote formats.

Suggested internal states include:

```text
CREATED
FUNDING_PENDING
SECURED
SETTLEMENT_AUTHORIZED
SETTLEMENT_PENDING
PAYOUT_PENDING
SETTLED
REFUND_AUTHORIZED
REFUND_PENDING
REFUNDED
DISPUTED
RECONCILIATION_REQUIRED
```

These are internal service states, not automatically canonical PIP-02 states.

Never expose a public endpoint that accepts an arbitrary payment hash and preimage or executes a raw settlement command. The execution worker must consume a durable, validated directive.

---

## 13. Testing and production readiness

### Protocol tests

Test valid and invalid Agent definitions, descriptors, roots, signatures, commitments, role bindings, predecessor links, forks, replays, authorization gates, disputes, resolution effects, expiry, and public/private separation.

### Hold-invoice tests

On regtest, prove pending funding, correct payment-hash/preimage behavior, authorized claim, permitted failure, HTLC expiry, restart recovery, ambiguous-result reconciliation, and safe handling of partial or multipart funding.

### Execution tests

Cover normal release, refund, disputed settlement/refund, unauthorized operations, duplicate directives, payout pending, payout failure, RPC timeout, crash before/after execution, HTLC expiry during dispute, late resolution, external failure without authorization, restart recovery, and fork detection.

### Production readiness

Do not use real-value trades until the implementation has:

* demonstrated the exact `lightning_hold_invoice` lifecycle on regtest;
* validated PIP-00, PIP-01, PIP-02 v2, and `pontmore/swap@1` versions;
* published the actual hold-invoice service schema;
* passed authorization, failure, expiry, restart, and reconciliation tests;
* undergone independent review of the Lightning construction and secret handling;
* documented the trusted-operator boundary and recovery limitations;
* updated [`README.md`](../README.md) whenever the implementation's custody model or protocol claims change.

---

## 14. Implementation milestones

1. **Read and align with README:** Confirm the implementation remains a node-controlled Lightning hold-invoice escrow as described in [`README.md`](../README.md).
2. **LDK feasibility:** Demonstrate the held-payment lifecycle on regtest.
3. **Lightning adapter:** Implement invoice creation, payment observation, claim/failure, payout tracking, and reconciliation.
4. **PIP-02 v2 validator:** Validate signatures, root binding, action linkage, authority, commitments, forks, expiry, and kernel invariants.
5. **PIP-01 descriptor/schema:** Publish the `lightning_hold_invoice` descriptor and actual versioned service schema.
6. **Swap profile integration:** Implement `pontmore/swap@1` terms, roles, actions, deadlines, disputes, and authorization gates.
7. **Durable execution journal:** Persist intent before Lightning operations and reconcile after restarts.
8. **Authorized release/refund:** Enforce profile/service authorization and verify external outcomes.
9. **Dispute and recovery integration:** Validate `core/open_dispute` and `core/resolve_dispute` effects without granting the resolver Lightning credentials.
10. **Interoperability and independent review:** Publish vectors and review the hold-invoice construction, authorization boundaries, secret management, and recovery behavior.

## License

MIT
