# Pontmore — LDK-Based HTLC Escrow

**A Pontmore-compatible Lightning escrow service using LDK, Nostr coordination, and an independently enforced escrow authorization policy.**

Pontmore HTLC Escrow is an experimental implementation of a node-controlled Lightning escrow service for peer-to-peer Bitcoin/fiat swaps. It connects Pontmore's public coordination protocol to a Lightning node and executes authorized settlement or refund operations.

The project separates three concerns:

* **Pontmore protocol:** Defines participant identities, escrow discovery, coordination history, authorization, and recovery rules.
* **Lightning node:** Manages Lightning payments and the underlying HTLC lifecycle.
* **Escrow daemon:** Enforces escrow-specific authorization, persists execution intent, reconciles payment outcomes, and publishes verified coordination results.

The initial implementation targets PIP-01 `custodial_escrow`, using a hold-invoice-based funding lock if the selected LDK implementation supports the required payment lifecycle.

The escrow operator controls the settlement capability and must be treated as trusted. Neither trading participant receives the escrow preimage or direct access to the node's settlement credentials.

This is an experimental proof of concept, not a claim of trustless or cryptographically enforced arbitration.

> **Protocol source of truth:** The canonical Pontmore protocol specifications and the exact versions pinned by a coordination determine interoperability. This README describes the intended implementation architecture; it does not override a PIP or a referenced service schema.

---

## 1. Protocol stack

```text
┌──────────────────────────────────────────────┐
│ PIP-00 — Agent Definition                    │
│ Public identity and capability discovery     │
└──────────────────────┬───────────────────────┘
                       │
┌──────────────────────▼───────────────────────┐
│ PIP-01 — Escrow Descriptor                   │
│ Kind 30361                                    │
│ Mechanism, networks, schema discovery         │
└──────────────────────┬───────────────────────┘
                       │
┌──────────────────────▼───────────────────────┐
│ PIP-02 — Coordination Event Chains           │
│ Kind 7300: immutable coordination root       │
│ Kind 7301: append-only coordination actions  │
└──────────────────────┬───────────────────────┘
                       │
┌──────────────────────▼───────────────────────┐
│ Pinned coordination profile                  │
│ pontmore/swap@1, when supported               │
│ Swap terms, roles, deadlines, recovery rules  │
└──────────────────────┬───────────────────────┘
                       │
┌──────────────────────▼───────────────────────┐
│ Versioned escrow service schema              │
│ OpenAPI / AsyncAPI                            │
│ API, authorization, idempotency, reconciliation│
└──────────────────────┬───────────────────────┘
                       │
┌──────────────────────▼───────────────────────┐
│ Pontmore Escrow Daemon                        │
│ Authorization · Persistence · Recovery       │
└──────────────────────┬───────────────────────┘
                       │
┌──────────────────────▼───────────────────────┐
│ LDK-based Lightning node                     │
│ Channels · HTLCs · Payment lifecycle         │
└──────────────────────────────────────────────┘
```

PIP-01 advertises compatibility and references a machine-readable service schema. PIP-02 defines a shared coordination kernel, while the pinned profile defines the application-specific swap lifecycle. Neither PIP substitutes for the Lightning implementation or the service's execution logic.

The PIP-02 version and swap-profile version must be pinned explicitly. Unsupported versions must be rejected rather than silently translated.

---

## 2. Custody model

This repository implements a **node-controlled escrow**, not an agent-controlled hold-invoice construction.

| Role            | Responsibility                                                                    |
| --------------- | --------------------------------------------------------------------------------- |
| `customer`      | Requests the swap and pays the escrow funding invoice.                            |
| `agent`         | Provides the agreed fiat payment and a Lightning payout invoice.                  |
| `core/escrow`   | Bound authority that confirms security and records settlement or refund.          |
| `core/resolver` | Bound authority that resolves a dispute when the pinned profile permits disputes. |
| Escrow daemon   | Validates authorization and executes the corresponding Lightning operation.       |
| LDK node        | Manages the Lightning payment and HTLC lifecycle.                                 |

The customer and agent are trading participants. The escrow operator is a separate authority.

### Custody boundary

The intended construction is:

1. The escrow service creates or arranges the funding payment condition.
2. The escrow service retains the settlement capability.
3. The customer funds the escrow through Lightning.
4. The escrow service verifies the incoming payment state.
5. A valid authorization permits release or refund.
6. The daemon executes the permitted operation and reconciles the result.

The preimage, if this hold-invoice construction requires the service to control it, is a secret. It must not be published to Nostr, sent to trading participants, or included in public evidence.

**Important:** A service-controlled preimage creates operator trust. Application-level authorization can make the honest implementation enforce policy, but cannot prevent a compromised or malicious operator from ignoring that policy.

---

## 3. Architecture

```text
                    CUSTOMER
                        │
                 Pays funding invoice
                        │
                        ▼
              ┌────────────────────┐
              │ LDK Lightning Node │
              │                    │
              │ Incoming payment   │
              │ HTLC / claim state │
              └─────────┬──────────┘
                        │
                        ▼
              ┌────────────────────┐
              │ Escrow Daemon      │
              │                    │
              │ Authorization      │
              │ Durable execution  │
              │ Recovery worker    │
              │ Payment reconcile  │
              └─────────┬──────────┘
                        │
                 Authorized outcome
                        │
                 ┌──────┴──────┐
                 │             │
              RELEASE         REFUND
                 │             │
          Settle incoming   Cancel/fail the
          payment, then    payment if the
          pay agent        actual state allows
                 │             │
                 └──────┬──────┘
                        │
                        ▼
                Verified outcome
                        │
                        ▼
                PIP-02 action chain
```

The diagram describes the desired lifecycle, not a guarantee that every LDK API exposes these exact operations. The implementation must first prove that the selected LDK stack can create or support the required held-payment construction.

### Component responsibilities

| Component        | Responsibility                                                                            |
| ---------------- | ----------------------------------------------------------------------------------------- |
| `lightning/`     | LDK integration, invoice handling, payment events, HTLC state and node operations.        |
| `escrow/`        | Funding validation, release/refund authorization and execution state.                     |
| `authorization/` | Validates participant, escrow and resolver authority against the root and pinned profile. |
| `coordination/`  | Validates PIP-02 roots and linked actions, reconstructs state, and detects forks.         |
| `nostr/`         | Signs and publishes public coordination events; handles private messages.                 |
| `storage/`       | Durable swap records, idempotency keys, execution intents and reconciliation checkpoints. |
| `recovery/`      | Reconciles incomplete operations without independently granting economic authority.       |

The escrow daemon must not equate a requested operation with a completed Lightning payment.

---

## 4. PIP-01 escrow descriptor

The public escrow descriptor is a kind `30361` addressable event. It declares the supported escrow type and networks and references a versioned service schema.

For this implementation, the intended declaration is:

* `escrow_type`: `lightning_hold_invoice`
* `networks`: `lightning`
* `service.schema.type`: `openapi` or `asyncapi`
* `service.schema.url`: an immutable or versioned HTTPS schema artifact
* `expires_at`: descriptor selection expiry
* `d` tag: stable descriptor identifier
* `t` tags: repeated `pontmore-network:lightning` filtering tag

Illustrative descriptor content:

```json
{
  "version": 1,
  "escrow_type": "lightning_hold_invoice",
  "networks": ["lightning"],
  "service": {
    "schema": {
      "type": "openapi",
      "url": "https://escrow.example.com/schemas/ldk-custodial-v1.json"
    }
  },
  "expires_at": 1780000000
}
```

This is an example, not a deployed descriptor. The service schema URL must resolve to the actual published schema before the descriptor is advertised as service-backed.

The descriptor MUST NOT contain:

* preimages or other settlement secrets;
* raw Lightning invoices;
* private payout instructions;
* API credentials or bearer tokens;
* private evidence or internal reconciliation records.

A descriptor does not prove that the service is online, solvent, safe, or trustworthy. Clients must validate the event signature, descriptor version, expiry, network compatibility, exact accepted event ID, and referenced schema.

---

## 5. PIP-02 coordination model

PIP-02 defines the coordination kernel and append-only event chain.

| Kind   | Purpose                       |
| ------ | ----------------------------- |
| `7300` | Immutable coordination root   |
| `7301` | Immutable coordination action |

The root binds the participants, authority identities, exact escrow descriptor revision, addressable descriptor coordinate, pinned profile and swap terms.

The action chain records state-changing claims through cryptographically linked actions. Each action references the root and its immediate predecessor. State is reconstructed by validating and replaying the chain, not by trusting relay arrival order or a database snapshot.

### Kernel actions

The escrow implementation must respect the meanings of the PIP-02 kernel actions:

* `core/accept`
* `core/decline`
* `core/secure`
* `core/authorize_settlement`
* `core/settle`
* `core/authorize_refund`
* `core/refund`
* `core/cancel`
* `core/expire`
* `core/open_dispute`
* `core/resolve_dispute`

These names are action values, not separate Nostr event kinds.

Swap-specific actions, including `swap/fiat_sent` and `swap/fiat_confirmed`, belong to the pinned swap profile, such as `pontmore/swap@1`, when supported.

The current published PIP-02 draft uses version `2` for its generalized root and action content. Earlier version-1 swap chains must be validated only under the version-1 specification supported by the implementation; versions must not be mixed within a coordination.

### Required invariants

The implementation must enforce at least these kernel rules:

* Exactly one immutable root and one canonical linear action chain.
* Authority is derived from the root and the pinned profile.
* Only the bound `core/escrow` authority can publish `core/secure`, `core/settle` or `core/refund`.
* Settlement requires prior authorization.
* Refund requires prior authorization or a valid dispute-resolution effect authorizing refund.
* Settlement and refund are mutually exclusive final economic outcomes.
* Ordinary economic progression is frozen during a dispute.
* A fork freezes further economic action; relay order, timestamp or event ID must not select a winner.
* Unsupported versions, duplicate actions, replays and invalid predecessors are rejected.

`core/resolve_dispute` records a permitted resolution effect. It does not itself move funds. The bound escrow authority must still execute and record the authorized outcome.

---

## 6. Swap lifecycle

The exact swap states and action permissions are defined by the pinned swap profile and the service schema. The following is the intended application-level flow, not a replacement for those specifications.

### Phase A — Request and acceptance

1. The customer and agent agree on the swap terms.
2. The customer publishes the immutable coordination root.
3. The root binds the agent, customer, escrow authority, resolver when required, exact PIP-01 descriptor event, descriptor coordinate and supported profile version.
4. The agent accepts or declines through a valid linked action.
5. Private payment details are exchanged through the profile-defined private channel.

The root's `expires_at` governs an unaccepted coordination. It is not automatically the funding invoice's HTLC expiry or the fiat deadline.

### Phase B — Funding

1. The escrow service creates the funding invoice or payment condition supported by its Lightning implementation.
2. The service generates and securely retains the settlement secret if the construction requires one.
3. The customer receives the invoice privately and pays it.
4. The daemon observes the actual payment and HTLC state.
5. The daemon validates the required funding conditions.
6. The bound escrow authority publishes `core/secure` only when the profile's security condition is met.

An invoice being created or delivered is not evidence that funding succeeded. An incomplete or partial funding attempt must be reconciled under the service schema.

### Phase C — Normal release

1. The agent performs the agreed fiat-side action.
2. The appropriate participant publishes the profile-defined claim, such as `swap/fiat_sent`.
3. The recipient confirms receipt if required by the profile.
4. The designated completion authority publishes `core/authorize_settlement` when all profile conditions are satisfied.
5. The escrow daemon validates the action chain and authorization.
6. The daemon durably records the execution intent and invokes the supported Lightning operation.
7. The daemon verifies the incoming settlement result.
8. It attempts the agent's outgoing payout and tracks that payment independently.
9. It publishes `core/settle` only when the final outcome meets the service schema and profile's completion conditions.

### Phase D — Refund and cancellation

A timeout, invoice expiry, or external recovery condition does not independently authorize `core/refund`.

The implementation must distinguish:

* request expiry before acceptance;
* funding failure before the escrow becomes secure;
* cancellation before a final economic outcome;
* an authorized refund;
* a Lightning payment that has actually failed or been canceled;
* an ambiguous or incomplete payment state.

When refund is authorized, the escrow daemon executes the operation supported by the actual Lightning state, verifies its outcome, and publishes `core/refund` only when the refunded outcome is established.

If the underlying payment state is ambiguous, the coordination must remain unresolved or enter the profile-defined recovery path. The implementation must not invent a final outcome.

### Phase E — Dispute resolution

1. An authorized participant opens a dispute using the profile-permitted `core/open_dispute` action.
2. Ordinary progression and economic action freeze.
3. The bound resolver evaluates evidence under the accepted policy and profile.
4. The resolver publishes a signed `core/resolve_dispute` action with a permitted effect: `resume`, `authorize_settlement`, `authorize_refund`, or `cancel`.
5. The escrow daemon validates the resolver's authority and the linked action.
6. If the effect authorizes settlement or refund, the daemon executes the corresponding operation and verifies the actual outcome.
7. The escrow authority publishes the final economic action only after reconciliation.

A resolver decision is an authorization record, not proof that Lightning settlement or cancellation succeeded.

---

## 7. LDK integration requirements

LDK is a toolkit, not a complete production node service. The implementation must choose and pin the exact LDK components and versions and provide the required persistence, networking, chain access and operational services.

Before advertising this escrow as operational, validate the following on Bitcoin regtest.

### Hold-invoice feasibility gate

* Can the chosen LDK stack create or support the exact incoming hold-invoice construction required?
* Can it retain an incoming payment in a pending state while the escrow waits for authorization?
* Can the application claim the payment only after the correct authorization?
* Can it fail or cancel the held payment safely when permitted?
* What happens at HTLC expiry, after a process restart, and when an RPC or event result is ambiguous?
* Can the application recover the actual payment state after a crash without duplicating a payout?

Do not assume that a normal invoice API or an ordinary inbound-payment claim callback automatically provides a complete hold-invoice escrow implementation.

If the selected LDK APIs cannot provide the required construction safely, the project must either implement and review the missing functionality at the appropriate layer or choose a different backend for the initial prototype. Do not claim compatibility until the test suite demonstrates the behavior.

### Node responsibilities

The Lightning adapter must provide a narrow interface to the escrow service for:

* invoice and payment-condition creation;
* payment and HTLC state observation;
* settlement and failure operations supported by the chosen implementation;
* outgoing payout creation and tracking;
* durable payment identifiers and reconciliation;
* startup recovery and operational health checks.

The authorization layer must not rely on an event callback alone as proof that a payment reached its final state.

---

## 8. Service schema and API boundary

The referenced PIP-01 service schema defines the actual interface. It must document:

* service version and transport;
* authentication and authorization;
* escrow instance creation and participant binding;
* invoice delivery and funding verification;
* release, refund and cancellation operations;
* evidence submission and resolver binding;
* supported dispute-resolution effects;
* exact timeout and recovery behavior;
* idempotency keys, errors and retry semantics;
* payment-state reconciliation;
* quote format and exact amounts where applicable.

Every operation that can move value must be authenticated, authorized, idempotent and recoverable after process failure.

The daemon should expose explicit execution states, such as:

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

These are internal implementation states. They are not automatically PIP-02 kernel actions or canonical protocol states. The service schema and pinned profile must define the mapping between internal states, profile actions and externally verified outcomes.

---

## 9. Timeouts, HTLC expiry and recovery

A Lightning HTLC has a finite protocol lifetime. A dispute cannot safely wait indefinitely for an outcome.

The service schema must distinguish at least:

* coordination acceptance expiry;
* funding-invoice validity;
* the accepted HTLC expiry and relevant CLTV constraints;
* fiat-payment deadline;
* fiat-confirmation deadline;
* dispute-opening deadline;
* dispute-resolution deadline;
* operational safety margin.

Wall-clock seconds and block-height or CLTV deltas are different units and must not be compared as though they were interchangeable.

The service must ensure that the permitted resolution window fits within the actual payment constraints, including sufficient time for execution and recovery. If that cannot be guaranteed, the trade must not be accepted with those terms.

### External custody reconciliation

External Lightning state does not itself authorize a Pontmore economic action.

In particular:

* HTLC expiry or failure must not independently authorize `core/refund`.
* An absent `swap/fiat_confirmed` action must not be treated as proof that fiat was not received.
* A timeout must not select a default winner unless the pinned profile explicitly defines the relevant authorization and recovery path.
* A node restart or an ambiguous operation result must trigger reconciliation, not a second unguarded execution.
* Recovery must preserve the distinction between external payment facts and protocol authorization.

If a payment is canceled externally, the daemon records the verified fact and follows the permitted recovery path. It must not falsely publish a refund authorization or claim that a disputed coordination became canonical merely because an HTLC expired.

---

## 10. Persistence and idempotency

The daemon must assume that processes can crash between any two operations.

Persist, at minimum:

* coordination root ID and pinned profile version;
* accepted descriptor event ID and descriptor coordinate;
* participant and authority bindings;
* validated action-chain tip;
* escrow instance reference;
* funding invoice reference and payment hash;
* encrypted settlement secret, where required;
* incoming payment state;
* outgoing payout invoice and payment state;
* authorization event ID and effect;
* durable execution intent and idempotency key;
* last successful reconciliation checkpoint.

Secrets must be encrypted at rest, excluded from logs and protected by strict process and filesystem permissions.

Before retrying a Lightning operation, reconcile the actual node state. Do not infer failure solely from a timeout or lost response. The database is an execution journal; it does not override the validated public action chain or the node's actual payment state.

---

## 11. Public and private data

### Public Nostr events

Public coordination events should contain only the minimum data needed to validate the coordination and reconstruct its state.

They must not contain:

* preimages;
* raw Lightning invoices;
* payment credentials;
* bank or mobile-money details;
* raw evidence documents or screenshots;
* private investigation notes;
* service secrets or node credentials.

### Private application channel

Invoices, payout instructions and sensitive evidence travel through a private, profile-defined channel, such as Nostr Gift Wrap where implemented.

Private messages are supplementary. They do not replace, reorder or override the public coordination chain. Where a public action depends on a private payload, the pinned profile must define how that payload is committed to and verified.

A signed Nostr event authenticates its publisher; it does not prove the truth of a fiat-payment claim or the success of a Lightning operation.

---

## 12. Security model and limitations

### Security goals

* Trading participants cannot directly invoke the escrow node's settlement or cancellation operations.
* Only a validated, authorized action can trigger the daemon's normal release or refund path.
* The resolver does not receive the settlement secret or Lightning credentials.
* Duplicate requests and replayed directives cannot trigger duplicate economic execution.
* Final protocol actions reflect verified external outcomes.
* Forks, ambiguous payment states and recovery failures freeze unsafe progression rather than selecting a winner automatically.

### Explicit limitations

**Trusted operator:** The operator controls the escrow execution path. A compromised or malicious operator may act outside the protocol's authorization policy.

**No atomic two-leg payment:** Settling the incoming payment and paying the agent are separate Lightning operations. If the incoming payment settles but the outgoing payout remains pending, the daemon must preserve `PAYOUT_PENDING`, retry safely and never claim the trade is fully settled prematurely.

**Finite HTLC lifetime:** Dispute and recovery must complete within the payment's operational constraints.

**Liquidity and HTLC slots:** Pending payments consume network resources. This construction is unsuitable for arbitrarily long settlement or dispute windows.

**LDK capability dependency:** Hold-invoice behavior must be demonstrated for the exact pinned implementation. It must not be inferred from generic Lightning payment support.

**Experimental protocol:** Pontmore PIP-02 is a draft, and its generalized event-chain design remains experimental pending independent implementation evidence. Pin versions and use shared test vectors.

**No automatic cryptographic arbitration:** Nostr signatures authenticate the resolver's decision but do not force a Lightning node to obey it. The node-controlled construction remains dependent on the operator's honest execution.

---

## 13. Future direction: stronger settlement enforcement

A future subtype could investigate a construction that reduces unilateral operator or participant control through an appropriate cryptographic conditional-payment mechanism.

Any such design must be evaluated against actual Lightning protocol support, supported node APIs, adversarial cases, timeout behavior and independent cryptographic review.

A proposed PTLC/adaptor-signature design must not be described as enforced merely because an adaptor signature or resolver key appears in the architecture. The complete construction must demonstrate who can settle, under which conditions, and how funds recover safely when a party or resolver disappears.

A stronger subtype should declare its distinct custody, authorization and recovery properties through PIP-01 and its referenced service schema.

---

## 14. Proposed project structure

```text
pontmore-ldk-escrow/
├── README.md
├── docs/
│   ├── ARCHITECTURE.md
│   ├── SECURITY_MODEL.md
│   ├── SERVICE_SCHEMA.md
│   ├── LDK_HOLD_INVOICE_FEASIBILITY.md
│   └── RECOVERY.md
├── schemas/
│   └── custodial-escrow-v1.openapi.yaml
├── src/
│   ├── lightning/
│   │   ├── adapter.rs
│   │   ├── payments.rs
│   │   └── reconciliation.rs
│   ├── escrow/
│   │   ├── lifecycle.rs
│   │   ├── authorization.rs
│   │   └── execution.rs
│   ├── coordination/
│   │   ├── root.rs
│   │   ├── action.rs
│   │   └── validation.rs
│   ├── nostr/
│   │   ├── events.rs
│   │   └── private_messages.rs
│   ├── recovery/
│   │   └── worker.rs
│   └── storage/
│       ├── models.rs
│       └── repository.rs
└── tests/
    ├── regtest/
    ├── authorization/
    ├── recovery/
    └── protocol-vectors/
```

This is a proposed organization. Modules should be introduced as implementation needs become concrete rather than treated as already implemented components.

---

## 15. Development roadmap

1. **Validate LDK feasibility:** Prove the required held-payment lifecycle on regtest before building the complete service.
2. **Build the Lightning adapter:** Expose a minimal, testable interface for invoice creation, payment observation, settlement/failure and payout tracking.
3. **Implement PIP-02 validation:** Validate signatures, root binding, exact predecessor linkage, profile versions, authority and kernel invariants.
4. **Publish the PIP-01 schema:** Define the exact service operations, authorization, timeout, recovery and idempotency rules.
5. **Implement the durable execution journal:** Persist intent before operations and reconcile after restarts.
6. **Implement release and refund:** Require explicit valid authorization, verify actual node outcomes and publish final actions only after reconciliation.
7. **Test dispute and fork handling:** Freeze economic execution on disputes and forks; recover only under the bound authority and pinned rules.
8. **Publish interoperability vectors:** Cover valid and invalid chains, authorization failures, duplicate actions, timeouts, node crashes, partial outcomes and recovery.
9. **Document the trust boundary:** Clearly distinguish operator-controlled escrow from any future participant-controlled or cryptographically enforced subtype.

## License

MIT
