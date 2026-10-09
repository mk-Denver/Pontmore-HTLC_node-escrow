# Pontmore HTLC Escrow — Implementation Guide

**Building a Pontmore-compatible Lightning escrow service with LDK, Nostr coordination, and explicit economic authorization.**

This guide describes a node-controlled Lightning escrow service for Pontmore peer-to-peer Bitcoin/fiat swaps. It is an implementation guide for the escrow service, not a replacement for the Pontmore protocol specifications or the referenced service schema.

The architecture separates three systems:

* **Pontmore protocol:** Defines Agent identity and capability discovery, escrow discovery, coordination roots and actions, authorization, expiry, dispute effects, and public/private boundaries.
* **LDK-based Lightning node:** Manages Lightning payments, HTLCs, and the underlying payment lifecycle.
* **Pontmore escrow daemon:** Validates protocol authorization, executes permitted operations, persists execution intent, and reconciles external payment outcomes.

The initial implementation targets PIP-01 `custodial_escrow` with a Lightning backend, subject to validating that the selected LDK implementation supports the required hold-invoice construction.

The escrow operator controls the settlement capability. Trading participants do not receive the escrow preimage or direct access to the node's settlement credentials.

This is an experimental proof of concept, not a trustless escrow construction.

> **Protocol source of truth:** The canonical [Pontmore protocol repository](https://github.com/pontmore/protocol) and the exact specification versions pinned by a coordination are authoritative. The current active series contains PIP-00, PIP-01, and PIP-02 only. Swap-specific semantics are supplied by the experimental `pontmore/swap@1` coordination profile, which is a profile and not a fourth PIP.

> **Alternative subtype:** An Agent-controlled `lightning_hold_invoice` construction is a separate security model. It must have its own descriptor, service schema, authorization rules, and recovery semantics.

---

## 1. Architecture and responsibilities

```text
Customer                         Agent
   │                               │
   │ Pays funding invoice          │ Provides payout invoice
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
│  Lightning payments · HTLC lifecycle      │
│  Channel and chain monitoring             │
└─────────────────────┬─────────────────────┘
                      │
                      ▼
               Lightning Network

Nostr coordination:
  PIP-00 Agent definition
       → PIP-01 escrow descriptor
       → PIP-02 v2 coordination root and action chain
       → pinned profile: pontmore/swap@1
       → referenced escrow service schema
```

### Component responsibilities

| Component | Responsibility |
| --- | --- |
| LDK node | Executes supported Lightning operations and reports payment state. |
| Escrow daemon | Enforces the service authorization policy and manages execution. |
| Authorization engine | Validates actors, actions, profile, deadlines, predecessor state, and accepted service rules. |
| Coordination validator | Validates the PIP-02 v2 root and linked action chain. |
| Nostr publisher | Signs and publishes permitted coordination events. |
| Private messaging | Transports invoices, payment instructions, and sensitive evidence. |
| Persistent storage | Stores execution intent, payment identifiers, authorization references, and reconciliation checkpoints. |
| Recovery worker | Reconciles actual Lightning state after timeouts, failures, and restarts. |
| Resolver | Reviews disputes under the accepted profile/service policy and publishes a permitted `core/resolve_dispute` effect; it receives no Lightning credentials. |

The daemon coordinates protocol state and execution. It must not treat a published Nostr event as proof that a Lightning payment succeeded.

---

## 2. Protocol dependencies

The implementation depends on these protocol resources:

| Resource | Purpose |
| --- | --- |
| [PIP-00 — Agent Definition](https://github.com/pontmore/protocol/blob/main/PIP-00-agent-definition.md) | Publishes and discovers versioned Agent capabilities and references to protocol resources. |
| [PIP-01 — Escrow Descriptor](https://github.com/pontmore/protocol/blob/main/PIP-01-escrow-descriptor.md) | Publishes the escrow mechanism, supported network, expiry, and optional service-schema reference. |
| [PIP-02 — Coordination Event Chains](https://github.com/pontmore/protocol/blob/main/PIP-02-coordination-event-chains.md), version 2 | Defines immutable coordination roots, linked actions, authority binding, authorization invariants, expiry, forks, disputes, and resolution effects. |
| [`pontmore/swap@1`](https://github.com/pontmore/protocol/blob/main/profiles/swap-v1.md) | Experimental swap coordination profile. Defines swap terms, roles, profile actions, deadlines, dispute classes, and settlement/refund gates. This is not a PIP. |
| Referenced service schema | Defines concrete escrow operations, authentication, evidence handling, resolver binding, timeout/recovery behavior, reconciliation, fees, and exact quote rules. |

There is **no PIP-03** in the current Pontmore protocol. Dispute classes and permitted resolution effects are not defined by a separate PIP. PIP-02 supplies the shared dispute kernel and `core/resolve_dispute`; the pinned coordination profile and accepted escrow service/schema supply domain-specific dispute and service behavior.

The implementation must pin exact supported PIP, profile, descriptor-event, commitment, quote, and service-schema versions. It must reject unsupported versions rather than silently translating them into an assumed state machine.

### PIP-02 v2 event model

PIP-02 version 2 defines only two coordination event kinds:

* Kind `7300`: immutable coordination root.
* Kind `7301`: immutable coordination action.

Evidence, disputes, and notes do not have separate canonical event kinds. Evidence is referenced by an action, disputes are represented by actions, and operational notes remain private or application-local.

Kernel actions include:

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

The `pontmore/swap@1` profile additionally defines `swap/fiat_sent` and `swap/fiat_confirmed`. These are profile actions, not new event kinds.

`core/resolve_dispute.data` must contain a policy identifier and one permitted effect: `resume`, `authorize_settlement`, `authorize_refund`, or `cancel`. A resolution effect is an authorization/state transition; it does not move funds. The escrow authority must still publish the corresponding final `core/settle` or `core/refund` action.

---

## 3. Roles and trust model

| Protocol role | UI alias | Responsibility |
| --- | --- | --- |
| `swap/customer` | Buyer | One application participant in the swap. |
| `swap/agent` | Seller | The other application participant and capability publisher where applicable. |
| `core/escrow` | Escrow operator | Confirms security and records final settlement or refund. |
| `core/resolver` | Resolver | Publishes an authorized dispute-resolution effect when the profile permits disputes. |

Buyer and seller are explanatory aliases only. The public protocol uses the namespaced roles bound in the coordination root.

For `pontmore/swap@1`, economic roles are derived from the root's `direction`:

* `fiat_to_btc`: `swap/customer` sends fiat; `swap/agent` provides Bitcoin.
* `btc_to_fiat`: `swap/agent` sends fiat; `swap/customer` provides Bitcoin.

### Trust assumptions

The operator-controlled design assumes that:

1. The escrow service securely manages the settlement capability.
2. The authorization engine validates the complete PIP-02 chain and pinned profile.
3. The execution service checks authorization before invoking Lightning operations.
4. The recovery worker reports verified external outcomes rather than inventing economic authorization.
5. The resolver does not have direct access to the settlement secret or Lightning credentials.

These controls constrain an honest implementation. They do not cryptographically prevent a malicious operator with sufficient node access from violating policy.

---

## 4. Environment setup

### 4.1 Recommended implementation stack

* Rust with a pinned, compatible LDK release.
* LDK components or LDK Node, depending on the required API surface.
* Bitcoin Core regtest for development.
* A maintained Nostr library for event validation, signing, and encryption.
* SQLite for a single-node prototype, or PostgreSQL for a multi-process service.
* A maintained secret-management solution for preimages and signing keys.
* A local Nostr relay for integration testing.

### 4.2 Node capabilities

Before implementing the escrow service, establish that the selected Lightning stack can safely support the intended payment construction. Required capabilities include:

* Creating or supporting the required incoming hold-invoice construction.
* Observing pending HTLCs and accepted payments.
* Withholding claim until the application authorizes it.
* Claiming or failing a held payment using the supported API.
* Persisting and recovering payment state across restarts.
* Tracking outgoing payout payments independently.
* Handling payment expiry, partial payments where supported, and ambiguous operation results.

**Do not assume that LDK's ordinary invoice or inbound-payment APIs automatically provide a complete hold-invoice workflow.** If the selected stack cannot support the required behavior, keep the implementation in feasibility testing or select another backend. Do not silently weaken the funding condition.

### 4.3 Development environment

Start with Bitcoin Core regtest, an LDK-based node with persistent storage, customer and Agent test wallets, a local Nostr relay, separate signing keys for the customer, Agent, escrow authority, and resolver, and a test database with an encrypted secret store.

The first milestone is proving the Lightning payment lifecycle, not building a user interface.

---

## 5. Nostr identity and private messaging

### 5.1 Key separation

Use separate identities or explicitly separated signing roles for the customer, Agent, escrow authority, and resolver. Each role signs only events permitted by PIP-02 and the pinned profile.

* Customer/Agent: publish the root and permitted participant/profile actions.
* Escrow authority: publish escrow-authorized security and economic outcomes.
* Resolver: publish `core/resolve_dispute` when authorized.

The resolver key is not a Lightning execution credential. Never store an `nsec` in plaintext configuration, source code, logs, or public repositories.

### 5.2 Relay subscriptions

Subscribe to event kinds and filters required by the pinned resources. At minimum, discover:

* Agent definitions and referenced escrow descriptors.
* Coordination roots referencing the escrow descriptor.
* Linked kind `7301` actions for active coordinations.
* Private messages addressed to the service.

Do not treat arbitrary events as trusted commands. Validate signatures, event references, role authorization, exact profile/version binding, descriptor references, and predecessor linkage. Relay arrival order and timestamps do not establish action order; the `prev` link does.

### 5.3 Private payloads

Private payloads may include funding invoices, Agent payout invoices, fiat payment instructions, delivery proofs, dispute evidence, and service references. Use a maintained Nostr implementation of NIP-44 and Gift Wrap where required by the selected private-messaging design.

Private messages supplement the public chain; they do not override it. A private message cannot independently authorize settlement or refund. Never place preimages, node credentials, raw private evidence, or payment secrets in public coordination events.

---

## 6. Publish the PIP-01 escrow descriptor

Before accepting swaps, the operator publishes a kind `30361` addressable escrow descriptor. The descriptor must identify the actual escrow subtype and reference the corresponding versioned service schema.

Example illustrative content:

```json
{
  "version": 1,
  "escrow_type": "custodial_escrow",
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

The actual event must include every field and tag required by the selected PIP-01 revision. `content.networks` is canonical; corresponding `pontmore-network:` tags are indexes and must not claim undeclared networks.

### Descriptor validation

The descriptor publisher must:

1. Use the addressable-event structure and stable `d` tag.
2. Declare the correct escrow type and network.
3. Reference an actual, versioned OpenAPI or AsyncAPI schema.
4. Use an absolute HTTPS schema URL.
5. Apply fetch-safety checks: reject private or unsafe destinations, unbounded redirects, and oversized responses.
6. Ensure the schema operations match the implementation.
7. Keep the descriptor current and publish replacements according to PIP-01 when contents change.

A coordination must bind both the descriptor coordinate and the exact descriptor event ID accepted. The descriptor advertises compatibility; it does not prove solvency, availability, security, or trustworthiness.

The referenced service schema owns service-specific behavior, including funding status, partial-funding recovery, release, refund, cancellation, idempotency, resolver binding, evidence submission, timeout/recovery, reconciliation, fees, and quote formats.

---

## 7. Create and validate the coordination root

For swap coordination, the customer or Agent creates a kind `7300` PIP-02 v2 root that pins `pontmore/swap@1` and binds:

* The swap terms and direction.
* `swap/customer` and `swap/agent` identities.
* Exactly one `core/escrow` authority.
* A `core/resolver` authority when disputes are enabled.
* The exact accepted descriptor event and its addressable coordinate.
* Profile and protocol versions.
* Relevant commitments, quotes, and deadlines.

The `pontmore/swap@1` terms include fiat direction, exact fiat amount and currency, Bitcoin amount/network, payment-channel identifier, and `fiat_pay_by`/`fiat_confirm_by` deadlines. The root's PIP-02 `expires_at` is the acceptance deadline and must precede `fiat_pay_by`.

### Request and acceptance flow

1. Customer and Agent negotiate terms and any private payloads.
2. The proposer publishes the immutable root.
3. The other application participant validates the root, Agent definition, descriptor, service schema, commitments, and profile.
4. The accepting participant publishes `core/accept` before root expiry, or publishes an allowed decline.
5. The escrow service validates the linked action chain.
6. Private payment details are exchanged through the accepted service channel.

A Nostr event has one author and one event signature. Do not describe a single request as doubly signed. Separate signed actions express consent by different participants.

Reject requests with an expired or invalid descriptor, unsupported PIP/profile version, invalid commitment, untrusted or incorrectly bound authority, invalid predecessor, or incompatible service schema.

---

## 8. HTLC funding lifecycle

### 8.1 Funding feasibility gate

Before production implementation, prove on regtest that the selected LDK stack can create or support the intended hold invoice, keep the payment pending until authorization, retain the settlement capability securely, claim or fail through supported APIs, recover after restart, and handle the actual HTLC expiry safely.

### 8.2 Invoice creation

Once validated:

1. Generate a cryptographically random preimage if the construction requires an application-generated preimage.
2. Compute the payment hash.
3. Create the hold invoice or payment condition through the backend's supported API.
4. Persist invoice reference, payment hash, amount, expiry data, and encrypted preimage where applicable.
5. Deliver the invoice privately.
6. Record internal funding state.
7. Publish only the profile-defined action appropriate to the verified security condition.

Do not copy LND-specific RPC commands into an LDK implementation and assume equivalent semantics.

### 8.3 Observe incoming payment

Reconcile supported LDK payment events with durable storage. When payment activity is observed:

1. Match it to the correct coordination and payment hash.
2. Validate amount and funding conditions.
3. Confirm that the observed state matches the payment construction.
4. Handle multipart payments according to the accepted service schema; a partial shard is not automatically complete funding.
5. Persist observed state.
6. Publish `core/secure` only when the Bitcoin security condition required by the profile and accepted service has been satisfied.

Invoice creation, delivery, and partial payment observation do not prove that funding is complete.

---

## 9. Agent payout invoice

The Agent provides a Lightning payout invoice through the private channel. Before release, validate its network, amount, fee treatment, expiry, swap binding, replay protection, liquidity feasibility, and compatibility with the accepted service schema.

The payout invoice is private. Public actions may contain only a permitted opaque reference or commitment. A valid payout invoice does not authorize release.

---

## 10. Authorization and economic execution

The execution service validates the current PIP-02 chain, profile state, accepted service rules, and required authorization before performing an economic operation.

### 10.1 Normal release

For `pontmore/swap@1`, the normal path is:

1. The fiat sender performs the agreed fiat-side obligation.
2. The fiat sender publishes `swap/fiat_sent` before `fiat_pay_by`.
3. The fiat receiver confirms receipt and publishes `swap/fiat_confirmed` before `fiat_confirm_by`.
4. The fiat receiver publishes `core/authorize_settlement`.
5. The daemon validates the signer, chain tip, profile condition, and service authorization.
6. It durably records an idempotent execution intent.
7. The escrow authority performs the supported Lightning claim/settlement operation.
8. The daemon reconciles the actual incoming payment result.
9. It attempts the Agent's outgoing payout.
10. The escrow authority publishes `core/settle` only when the required final outcome is established under the accepted service schema.

Under `pontmore/swap@1`, `core/authorize_settlement` is authorized by the fiat receiver, not automatically by the resolver.

### 10.2 Two-leg settlement is not atomic

Incoming escrow settlement and outgoing payout are separate operations. Persist the incoming result, track the payout independently, prevent duplicate payouts, reconcile before retrying, request a replacement invoice only if the service schema permits it, and avoid publishing a completed outcome prematurely. A successful incoming claim does not prove that the Agent received the payout.

### 10.3 Refund authorization

Under `pontmore/swap@1`, the Bitcoin provider may publish `core/authorize_refund` after `fiat_pay_by` elapses without a valid `swap/fiat_sent`, subject to the profile and accepted service rules. The escrow authority then publishes `core/refund` after executing and reconciling the permitted refund operation.

A timeout, HTLC expiry, node restart, or external recovery condition is not automatically a Pontmore refund authorization. If a dispute resolution produces `authorize_refund`, it supplies the permitted authorization effect; it does not itself move funds or replace the escrow authority's final `core/refund` action.

### 10.4 Cancellation and expiry

Cancellation is distinct from refund. `core/cancel` is used only when permitted before a final economic outcome, including an allowed dispute-resolution effect. `core/expire` records a profile-defined expiry and recovery condition; it is not a general substitute for refund. Never map every timeout or payment failure to `core/cancel` or `core/refund` without checking the profile, service schema, and current payment state.

---

## 11. Dispute resolution

PIP-02 supplies the dispute kernel. The pinned profile supplies domain dispute classes and permitted effects, while the accepted escrow service/schema supplies evidence handling, resolver binding, service operations, and evaluation policy. No separate PIP-03 is involved.

### 11.1 Opening a dispute

A participant publishes `core/open_dispute` when authorized by PIP-02 and `pontmore/swap@1`. The daemon validates identity, root and chain tip, stage, deadlines, profile-defined dispute class, and accepted service requirements.

Opening a valid dispute freezes ordinary profile progress, settlement authorization, settlement, and refund until a valid `core/resolve_dispute` effect is recorded. Do not continue normal economic execution while disputed unless the protocol explicitly permits the relevant recovery path.

### 11.2 Evidence submission

Evidence is submitted through the profile/service-defined private channel. It may include fiat receipts, delivery confirmations, signed messages, oracle attestations, timestamps, and references. Evidence must be authenticated and bound to the correct coordination. A screenshot, signed claim, or evidence reference is not automatically proof of the underlying transaction.

Public actions should expose only the minimum facts required by the protocol. Use PIP-02 evidence references, commitments, or opaque service references rather than raw invoices, account details, receipts, or credentials.

### 11.3 Resolver decision

The resolver publishes `core/resolve_dispute` with the required `policy` and permitted `effect`:

* `resume`: return to the pre-dispute state.
* `authorize_settlement`: derive settlement authorization; the escrow authority must still publish `core/settle`.
* `authorize_refund`: derive refund authorization; the escrow authority must still publish `core/refund`.
* `cancel`: derive terminal cancellation.

The daemon verifies resolver identity, event signature, root binding, predecessor, current disputed state, policy binding, permitted effect, deadlines, and replay protection. A resolution effect is not a Lightning payment.

### 11.4 Resolver/executor separation

The resolver does not receive the preimage or Lightning credentials. It signs a policy decision; the execution service independently validates that decision and the actual Lightning state before acting.

---

## 12. Timeout policy and external custody reconciliation

PIP-02 and `pontmore/swap@1` define protocol-level expiry and recovery conditions. The accepted service schema defines concrete service timeout and reconciliation behavior. Lightning defines the actual payment and HTLC lifetime. These are related but distinct layers.

Distinguish at least:

* Root acceptance expiry (`expires_at`).
* Descriptor expiry.
* Funding invoice validity.
* Actual accepted HTLC expiry and relevant CLTV constraints.
* `fiat_pay_by` and `fiat_confirm_by`.
* Dispute-opening and service-resolution deadlines.
* Execution and recovery safety buffers.

Wall-clock seconds and block-height/CLTV values are different units and must not be compared directly.

### Recovery invariants

The daemon must enforce that:

* External payment state does not itself authorize a Pontmore economic action.
* HTLC expiry or failure does not independently authorize `core/refund`.
* Absence of `swap/fiat_confirmed` is not proof that fiat was not received.
* A timeout does not select a default winner unless the pinned profile and service explicitly define the authorization and recovery path.
* A lost RPC response is not proof that an operation failed.
* A restart does not authorize replaying an economic operation.
* An ambiguous payment state must be reconciled before final publication.
* A fork or invalid action chain freezes unsafe economic progression.

If Lightning confirms that a payment failed or expired, record that verified external fact and apply the valid service/profile recovery path. Do not fabricate an authorization action. If the outcome is unknown, mark execution for reconciliation and avoid publishing a false final state.

---

## 13. Persistent storage and idempotency

The service must survive crashes between authorization, execution, reconciliation, and publication.

Persist at least:

### Swap record

* Coordination root ID and exact descriptor event ID/coordinate.
* Customer and Agent identities.
* Bound escrow and resolver authorities.
* Pinned PIP-02 and profile versions.
* Terms, commitments, quote references, and deadlines.
* Funding invoice reference and payment hash.
* Encrypted preimage where required.
* Actual accepted HTLC expiry.
* Current validated action-chain tip and derived state.
* Current internal execution state.

### Authorization record

* Authorizing action event ID.
* Action and permitted effect.
* Authorized actor and authority binding.
* Validated predecessor.
* Applicable deadline.
* Idempotency key.
* Validation result.

### Execution record

* Swap ID and operation type.
* Durable execution intent.
* Incoming and outgoing payment references.
* Attempt count and status.
* Last reconciled node state.
* Last error and recovery checkpoint.
* Final verified outcome, if any.

Use transactions, compare-and-swap updates, or equivalent concurrency controls. Before retrying any Lightning operation, reconcile actual node state. Never infer failure solely from a request timeout.

The database is an execution journal. It does not override the validated public chain or actual Lightning state.

---

## 14. Service API

The exact API must be defined in the versioned OpenAPI or AsyncAPI artifact referenced by the PIP-01 descriptor. A service may expose operations equivalent to:

| Operation | Purpose |
| --- | --- |
| Create escrow | Create a swap-bound service instance. |
| Get escrow status | Retrieve reconciled service and payment state. |
| Get funding instructions | Deliver a private funding invoice or supported alternative. |
| Submit payout invoice | Register or replace the Agent invoice where permitted. |
| Submit evidence | Submit authenticated evidence through the supported private channel. |
| Get execution status | Retrieve reconciled incoming and outgoing payment states. |
| Request reconciliation | Trigger authenticated reconciliation of external payment state. |

The API must define authentication, authorization, errors, idempotency, versioning, rate limits, privacy, resolver binding, and recovery behavior.

**Never expose a public endpoint that accepts an arbitrary payment hash and preimage or executes a raw settlement command.** The execution worker must consume a durable, validated directive, and node operations must be isolated from public API processes.

---

## 15. Testing

### 15.1 Protocol tests

* Valid and invalid PIP-00 Agent definitions.
* Invalid PIP-01 descriptor references and expired descriptors.
* Unsupported PIP-02 v2 or `pontmore/swap@1` versions.
* Invalid root signatures, role bindings, commitments, and predecessors.
* Unauthorized actors and replayed actions.
* Conflicting forks and frozen economic progression.
* Duplicate authorizations.
* Settlement without prior authorization.
* Refund without prior authorization or a valid resolution effect.
* Invalid `core/resolve_dispute` policy/effect data.
* Dispute freezing ordinary progress.
* `resume`, `authorize_settlement`, `authorize_refund`, and `cancel` resolution effects.
* Timeout with no valid economic authorization.
* Public/private data separation.

### 15.2 Lightning feasibility tests

On regtest, prove that the selected LDK stack supports the required hold-invoice construction, pending funding, settlement-secret matching, claim/failure operations, expected expiry behavior, restart recovery, safe failure of unsupported modes, and the declared partial/multipart funding policy.

### 15.3 Execution tests

* Normal release after valid `swap/fiat_confirmed` and `core/authorize_settlement`.
* Refund after valid profile authorization.
* Disputed settlement and refund authorization through `core/resolve_dispute`.
* Unauthorized settlement/refund attempts.
* Duplicate execution directives.
* Incoming settlement success with payout pending.
* Outgoing payout failure and safe retry.
* RPC timeout with unknown outcome.
* Crash before and after execution intent.
* Crash after execution but before final publication.
* HTLC expiry during a dispute.
* Late dispute or resolution that cannot complete safely.
* External payment failure without protocol authorization.
* Recovery after daemon restart.
* Fork discovered while execution is pending.

### 15.4 End-to-end environment

Use Bitcoin Core regtest, the selected LDK implementation, customer and Agent wallets, a local Nostr relay, separate test identities, a persistent database, and automated assertions against both the public PIP-02 action chain and actual Lightning payment state.

A successful test must verify protocol correctness and the external payment result. A final Nostr event alone is not proof that funds moved as intended.

---

## 16. Security and deployment

### Key and secret management

* Encrypt Nostr signing keys at rest.
* Encrypt the preimage where the construction requires the service to retain it.
* Separate resolver signing credentials from Lightning execution credentials.
* Keep node credentials out of the public API and general application workers.
* Redact invoices, preimages, private evidence, and credentials from logs.
* Limit production-secret access and audit sensitive operations.

### Operational monitoring

Alert on pending HTLCs approaching safety deadlines, disputes approaching service deadlines, incoming settlement with payout pending, reconciliation failures, repeated or conflicting authorizations, relay disconnections, database or secret-store failures, unexpected payment transitions, and mismatches between the public chain and external payment state.

### Operator accountability

Signed actions and durable execution records make it possible to audit intended policy and reported outcomes. They do not prevent a malicious or compromised operator from using its Lightning credentials outside protocol policy.

### Production readiness

Do not deploy real-value trades until the implementation has demonstrated the required hold-invoice behavior on regtest, validated exact PIP-00/PIP-01/PIP-02 v2 and `pontmore/swap@1` revisions, published a complete service schema, passed authorization/failure/restart/reconciliation tests, undergone independent review of the Lightning construction and secret handling, and documented the operator trust and recovery limitations.

---

## 17. Future direction: stronger settlement enforcement

The current design is operator-controlled. A future construction may investigate whether an independently reviewed cryptographic payment mechanism can reduce unilateral settlement control.

Any PTLC or adaptor-signature proposal must define and prove who can settle, which secret or signature enables settlement, what happens when the resolver disappears, whether either participant can settle unilaterally, how disputes constrain economic progression, how timeout and recovery operate, and whether the selected Lightning implementation supports the construction.

Do not advertise a PTLC/adaptor-signature design as cryptographically enforced merely because it includes an adaptor signature or resolver key. A complete construction must establish the claimed security properties and advertise its distinct custody, authorization, timeout, and recovery guarantees through a separate PIP-01 subtype and versioned service schema.

---

## 18. Implementation milestones

1. **LDK feasibility:** Demonstrate the required held-payment lifecycle on regtest.
2. **Lightning adapter:** Build a narrow interface for payment observation, claim/failure, payout tracking, and reconciliation.
3. **PIP-02 v2 validator:** Validate signatures, root binding, linked actions, authority, commitments, forks, expiry, and supported versions.
4. **PIP-01 service schema:** Publish the concrete service API and authorization contract.
5. **Swap profile integration:** Implement and test `pontmore/swap@1` terms, roles, actions, deadlines, and authorization gates.
6. **Durable execution journal:** Persist intent before operations and reconcile after restarts.
7. **Authorized release/refund:** Enforce profile/service authorization and verify external outcomes.
8. **Dispute integration:** Validate `core/open_dispute` and resolver effects from `core/resolve_dispute`.
9. **Recovery testing:** Cover expiry, ambiguous results, crashes, forks, and partial settlement.
10. **Interoperability tests:** Publish shared vectors for supported PIP-02 v2 and `pontmore/swap@1` behavior.
11. **Independent review:** Review the Lightning construction, authorization boundaries, secret management, and recovery behavior.

## License

MIT
