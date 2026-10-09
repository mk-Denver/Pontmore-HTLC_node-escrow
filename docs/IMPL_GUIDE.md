# Pontmore HTLC Escrow — Implementation Guide

**Building a Pontmore-compatible Lightning escrow service with LDK, Nostr coordination, and explicit economic authorization.**

This guide describes the implementation of a node-controlled Lightning escrow service for Pontmore peer-to-peer Bitcoin/fiat swaps.

The architecture separates three systems:

* **Pontmore protocol:** Defines identities, escrow discovery, coordination history, authorization, and dispute policy.
* **LDK-based Lightning node:** Manages Lightning payments, HTLCs, and the underlying payment lifecycle.
* **Pontmore escrow daemon:** Validates authorization, executes permitted operations, persists execution intent, and reconciles external payment outcomes.

The initial implementation targets PIP-01 `custodial_escrow` with a Lightning backend, subject to validating that the selected LDK implementation supports the required hold-invoice construction.

The escrow operator controls the settlement capability. Trading participants do not receive the escrow preimage or direct access to the node's settlement credentials.

This is an experimental proof of concept, not a trustless escrow construction.

> **Protocol source of truth:** The canonical [Pontmore protocol repository](https://github.com/pontmore/protocol) and the exact specification versions pinned by a swap are authoritative. This guide describes implementation behavior and does not override a PIP or a referenced service schema.

> **Alternative subtype:** An agent-controlled `lightning_hold_invoice` construction is a separate security model. It must have its own descriptor, service schema, authorization rules, and recovery semantics. Do not advertise the operator-controlled implementation as participant-controlled merely because it uses a Lightning hold invoice.

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
  PIP-01 descriptor → PIP-02 action chain → pinned swap profile
                                           → PIP-03 dispute policy
```

### Component responsibilities

| Component              | Responsibility                                                                                          |
| ---------------------- | ------------------------------------------------------------------------------------------------------- |
| LDK node               | Executes supported Lightning operations and reports payment state.                                      |
| Escrow daemon          | Enforces the service's authorization policy and manages execution.                                      |
| Authorization engine   | Validates the actor, action, profile, deadlines, and predecessor state.                                 |
| Coordination validator | Validates the PIP-02 root and action chain.                                                             |
| Nostr publisher        | Signs and publishes permitted coordination events.                                                      |
| Private messaging      | Transports invoices, payment instructions, and sensitive evidence.                                      |
| Persistent storage     | Stores execution intent, payment identifiers, authorization references, and reconciliation checkpoints. |
| Recovery worker        | Reconciles actual Lightning state after timeouts, failures, and restarts.                               |
| Resolver/solver        | Reviews disputes and publishes an authorized resolution; receives no Lightning credentials.             |

The daemon coordinates the protocol and execution. It must not treat a published Nostr event as proof that a Lightning payment has succeeded.

---

## 2. Protocol dependencies

The implementation depends on the following protocol layers.

| Specification                      | Purpose                                                                            |
| ---------------------------------- | ---------------------------------------------------------------------------------- |
| PIP-00 — Agent Definition          | Advertises agent identity, capabilities, and trading terms.                        |
| PIP-01 — Escrow Descriptor         | Advertises the escrow type, network, and service schema.                           |
| PIP-02 — Coordination Event Chains | Defines the coordination root, linked actions, authority, and chain validation.    |
| PIP-03 — Dispute Policy            | Defines dispute and timeout policy constraints.                                    |
| Pinned swap profile                | Defines swap-specific terms, actions, and authorization conditions.                |
| Referenced service schema          | Defines the concrete escrow API, authentication, execution, and recovery behavior. |

The implementation must pin supported specification and profile versions. It must reject unsupported versions rather than silently translate them into an assumed state machine.

### PIP-02 event model

The current generalized PIP-02 draft uses:

* Kind `7300` for the immutable coordination root.
* Kind `7301` for append-only coordination actions.

Kernel actions such as `core/accept`, `core/secure`, `core/authorize_settlement`, `core/settle`, `core/authorize_refund`, `core/refund`, and `core/resolve_dispute` are action values, not separate event kinds.

Swap-specific actions, including `swap/fiat_sent` and `swap/fiat_confirmed`, must be interpreted according to the pinned swap profile.

Do not assume that dispute, evidence, or note event kinds from an earlier profile are part of the current canonical kernel. Use the exact event grammar of the pinned specifications.

---

## 3. Roles and trust model

| Protocol role             | UI alias        | Responsibility                                  |
| ------------------------- | --------------- | ----------------------------------------------- |
| `customer`                | Buyer           | Requests the swap and funds the escrow.         |
| `agent`                   | Seller          | Provides the traded claim and a payout invoice. |
| `core/escrow` authority   | Escrow operator | Executes authorized economic outcomes.          |
| `core/resolver` authority | Resolver/solver | Issues a permitted dispute-resolution effect.   |

The public protocol uses canonical role names. Buyer and seller are explanatory UI aliases only.

### Trust assumptions

The operator-controlled design assumes that:

1. The escrow service securely manages the settlement capability.
2. The authorization engine validates the complete action chain.
3. The execution service checks authorization before invoking Lightning operations.
4. The recovery worker reports verified external outcomes rather than inventing economic authorization.
5. The resolver does not have direct access to the settlement secret or Lightning credentials.

These controls constrain the intended behavior of an honest implementation. They do not cryptographically prevent a malicious operator with sufficient node access from violating policy.

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

Before implementing the escrow service, establish that the selected Lightning stack can safely support the intended payment construction.

The required capabilities include:

* Creating or supporting the required incoming hold-invoice construction.
* Observing pending HTLCs and accepted payments.
* Withholding claim until the application authorizes it.
* Claiming or failing a held payment using the supported API.
* Persisting and recovering payment state across restarts.
* Tracking outgoing payout payments independently.
* Handling payment expiry, partial payments where supported, and ambiguous operation results.

**Do not assume that LDK's ordinary invoice or inbound-payment APIs automatically provide a complete Mostro-style hold-invoice workflow.**

If the selected LDK stack cannot support the required behavior, implement and review the missing functionality at the appropriate layer or select a different backend for the initial prototype. Do not claim the escrow subtype is operational until the required behavior is demonstrated.

### 4.3 Development environment

Start with:

1. Bitcoin Core in regtest mode.
2. An LDK-based node with persistent storage.
3. A customer wallet or test node.
4. An agent wallet or test node.
5. A local Nostr relay.
6. Separate customer, agent, escrow, and resolver signing keys.
7. A test database and encrypted secret store.

The first milestone is proving the Lightning payment lifecycle, not building the full user interface.

---

## 5. Nostr identity and private messaging

### 5.1 Key separation

Use separate identities or explicitly separated signing roles for the customer, agent, escrow authority, and resolver.

Each role signs only the events permitted by the pinned protocol and profile.

* Customer: publishes the coordination root and permitted participant actions.
* Agent: accepts or declines and publishes permitted participant actions.
* Escrow authority: publishes escrow-authorized security and economic outcomes.
* Resolver: publishes a permitted dispute-resolution effect when authorized.

A resolver's key must not be treated as a Lightning execution credential.

Never store an `nsec` in plaintext configuration, source code, logs, or public repositories. Prefer encrypted key storage or a suitable signing service.

### 5.2 Relay subscriptions

The application subscribes to the event kinds and filters defined by its pinned PIP versions.

At minimum, the implementation needs to discover:

* Coordination roots referencing its escrow descriptor.
* Linked actions associated with active swaps.
* The relevant agent definition and escrow descriptor.
* Private messages addressed to the service.
* Dispute and resolution information required by the selected profile.

Do not subscribe to arbitrary events and treat their content as trusted commands. Validate signatures, event references, role authorization, profile versions, and predecessor linkage.

### 5.3 Private payloads

Private payloads may include:

* Funding invoices.
* Agent payout invoices.
* Fiat payment instructions.
* Delivery proofs.
* Dispute evidence.
* Internal references needed for execution.

Use a maintained Nostr implementation of NIP-44 and Gift Wrap where required by the chosen private messaging design.

Private messages supplement the public coordination chain; they do not override it. A private message must not independently authorize a settlement or refund.

Never place preimages, node credentials, raw private evidence, or payment secrets in public coordination events.

---

## 6. Publish the PIP-01 escrow descriptor

Before accepting swaps, the operator publishes a kind `30361` escrow descriptor.

The descriptor must identify the actual escrow subtype and reference the corresponding versioned service schema.

For the operator-controlled implementation, the intended declaration is:

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

This is illustrative content, not a complete or deployed descriptor. The actual event must include every field and tag required by the selected PIP-01 revision.

### Descriptor validation

The descriptor publisher must:

1. Use the correct addressable-event structure and stable `d` tag.
2. Declare the correct escrow type and network.
3. Reference an actual, versioned OpenAPI or AsyncAPI schema.
4. Publish the corresponding discovery tags required by PIP-01.
5. Ensure the schema URL uses HTTPS and is safe to retrieve.
6. Avoid private or unsafe destinations, unbounded redirects, and oversized responses.
7. Ensure the schema's service operations match the implementation.
8. Keep the descriptor current and publish a replacement according to PIP-01's rules when its contents change.

The descriptor advertises compatibility. It does not prove that the service is solvent, available, secure, or trustworthy.

### Funding cardinality is not spending authority

If the selected descriptor version uses `funding_rules`, the fields describe the funding conditions defined by that specification.

A field such as `funding_threshold` must not be interpreted as a spending threshold, preimage threshold, resolver quorum, or settlement authorization policy unless the applicable PIP explicitly defines that meaning.

---

## 7. Create and validate the coordination root

The customer creates the immutable PIP-02 coordination root with the required canonical fields.

The root must bind:

* Swap identifier and swap type.
* Customer and agent identities.
* Exact escrow descriptor revision.
* Escrow authority.
* Resolver authority when required.
* Pinned profile and supported versions.
* Agreed swap terms.
* Relevant expiry and policy references.

The exact content and tags must match the selected PIP-02 and profile schemas.

### Request and acceptance flow

1. Customer and agent negotiate the terms.
2. Customer publishes the coordination root.
3. The agent validates the root, agent definition, escrow descriptor, and referenced service schema.
4. The agent publishes a permitted acceptance or decline action.
5. The escrow service validates the linked action chain.
6. Private payment details are exchanged through the declared private channel.

A Nostr event has one author and one event signature. Do not describe a single request as doubly signed. Separate signed actions express consent by different participants.

The escrow daemon must reject requests that reference an expired descriptor, unsupported profile, untrusted authority, invalid predecessor, or incompatible service schema.

---

## 8. HTLC funding lifecycle

### 8.1 Funding feasibility gate

Before building the production lifecycle, implement a minimal regtest experiment that answers:

1. Can the selected LDK stack create or support the intended hold invoice?
2. Can the incoming payment remain pending until explicit application authorization?
3. Can the service securely retain the settlement capability?
4. Can it claim or fail the payment using supported APIs?
5. Can it recover payment state after a process crash?
6. Can it handle the payment's actual HTLC expiry safely?

If any answer is unknown, keep the implementation in the feasibility stage.

### 8.2 Invoice creation

Once the required construction has been validated:

1. Generate the payment preimage using a cryptographically secure random-number generator if the selected construction requires an application-generated preimage.
2. Compute the corresponding payment hash.
3. Create the hold invoice or payment condition using the selected backend's supported API.
4. Persist the invoice reference, payment hash, expected amount, expiry information, and encrypted preimage where applicable.
5. Deliver the invoice privately to the customer.
6. Record the internal funding state.
7. Publish the relevant profile-defined coordination action.

The exact invoice-generation procedure is backend-specific. Do not copy LND-specific RPC commands into an LDK implementation and assume equivalent semantics.

### 8.3 Observe incoming payment

The daemon subscribes to or polls the supported LDK payment events and reconciles them with durable storage.

When a payment becomes pending:

1. Match it to the correct escrow instance and payment hash.
2. Validate the expected amount and funding conditions.
3. Verify that the observed state is consistent with the underlying payment construction.
4. For multipart payments, verify the required complete funding condition rather than treating a partial shard as a fully funded escrow.
5. Persist the observed payment state.
6. Publish `core/secure` only when the pinned profile's security condition has been satisfied.

Invoice creation, invoice delivery, and partial payment observation do not prove that escrow funding is complete.

If the selected LDK implementation does not support the required multipart or hold-invoice behavior, reject or disable that mode rather than silently weakening the funding conditions.

---

## 9. Agent payout invoice

The agent provides a Lightning payout invoice through the private channel.

Before release, the daemon validates:

* Invoice network and destination policy.
* Amount and fee treatment.
* Invoice expiry and remaining execution window.
* Binding to the correct swap.
* Duplicate-use and replay protection.
* Whether the invoice can be paid using the node's available liquidity.
* Whether the payout operation is compatible with the declared service schema.

The payout invoice is private. Public evidence may contain a permitted opaque reference or commitment, but must not expose the raw invoice.

A valid payout invoice does not itself authorize release.

---

## 10. Authorization and economic execution

The execution service validates the current action chain and the authorization required by the pinned profile before performing an economic operation.

### 10.1 Normal release

The normal release flow is:

1. The agent performs the agreed fiat-side obligation.
2. The relevant participant publishes the profile-defined claim, such as `swap/fiat_sent`.
3. Receipt is confirmed according to the pinned profile.
4. The appropriate authority publishes `core/authorize_settlement`.
5. The daemon validates the authorization and current chain tip.
6. It durably records an idempotent execution intent.
7. It invokes the supported Lightning claim or settlement operation.
8. It reconciles the actual incoming payment result.
9. It attempts the agent's outgoing payout.
10. It publishes the final economic outcome only when the conditions defined by the service schema and profile are satisfied.

The actual LDK API call depends on the validated payment construction.

### 10.2 Two-leg settlement is not atomic

The incoming escrow settlement and outgoing agent payout are separate operations.

The incoming payment may settle successfully while the payout remains pending or fails. The daemon must:

* Persist the incoming settlement result.
* Record the outgoing payout's independent status.
* Expose an internal `PAYOUT_PENDING` state where appropriate.
* Retry only after reconciling the outgoing payment.
* Prevent duplicate payouts.
* Request a replacement invoice if the service schema permits it.
* Avoid publishing a completed swap outcome prematurely.

A successful incoming claim does not prove that the agent received the payout.

### 10.3 Refund authorization

The daemon must validate the applicable authorization before initiating a refund.

Depending on the pinned profile, valid authorization may come from an explicit `core/authorize_refund` action or a valid dispute-resolution effect that authorizes a refund.

A timeout, HTLC expiry, node restart, or external recovery condition is not automatically a Pontmore refund authorization.

If the payment has already failed or expired, the daemon records the verified external fact and follows the permitted recovery path. It must not fabricate a refund operation or publish a final economic outcome unsupported by the actual payment state.

### 10.4 Cancellation

Cancellation is distinct from refund.

`core/cancel` must be used only under the circumstances permitted by the pinned protocol and profile. The daemon must distinguish a valid pre-economic cancellation from a refund of a secured escrow.

Never map every timeout or payment failure to `core/cancel` or `core/refund` without checking the applicable authorization and current payment state.

---

## 11. Dispute resolution

### 11.1 Opening a dispute

A participant opens a dispute through the action permitted by the pinned PIP-02 and profile.

The daemon validates:

* The participant's identity and authority.
* The root and current action-chain tip.
* Whether a dispute may be opened at this stage.
* The applicable dispute deadline.
* The selected PIP-03 policy.
* Any service-specific requirements.

Once a valid dispute freezes ordinary progression, the daemon must not continue normal economic execution unless the protocol explicitly permits the relevant action.

### 11.2 Evidence submission

The participants submit evidence through the profile-defined private channel.

Examples include:

* Fiat payment receipts.
* Delivery confirmations.
* Signed messages.
* Agreed oracle attestations.
* Relevant timestamps and references.

Evidence must be authenticated and bound to the correct swap. A screenshot or signed claim may support a decision but does not automatically prove the underlying fiat transaction occurred.

Public events should expose only the minimum information required by the protocol.

### 11.3 Resolver decision

The resolver evaluates the evidence and publishes the permitted PIP-02 dispute-resolution action.

Depending on the pinned PIP-02 revision, the resolution effect may authorize settlement, authorize refund, resume the permitted workflow, or cancel where allowed.

The daemon must verify:

1. Resolver identity and binding to the root.
2. Signature and event validity.
3. Action-chain predecessor and current state.
4. Permitted resolution effect.
5. Applicable deadline and profile rules.
6. Idempotency and replay protection.

A resolution effect is an authorization record, not a Lightning payment.

The daemon executes the authorized operation through the Lightning backend, reconciles the actual result, and publishes the permitted final economic action only after the outcome is established.

### 11.4 Solver/executor separation

The resolver does not receive the preimage or Lightning credentials.

The resolver signs a policy decision. The execution service independently validates that decision before acting.

This limits the consequences of a compromised resolver key, but it does not eliminate the trust placed in the escrow operator.

---

## 12. Timeout policy and external custody reconciliation

A hold-invoice HTLC has a finite lifetime. The implementation must not assume that a dispute can remain open indefinitely.

The service schema and pinned profile must distinguish:

* Coordination expiry.
* Funding invoice validity.
* Actual accepted HTLC expiry and relevant CLTV constraints.
* Fiat-payment deadline.
* Fiat-confirmation deadline.
* Dispute-opening deadline.
* Dispute-resolution deadline.
* Execution and recovery safety buffer.

Wall-clock seconds and block-height or CLTV values are different units. Do not compare them directly.

### Recovery invariants

The daemon must enforce these invariants:

* External payment state does not itself authorize a Pontmore economic action.
* HTLC expiry or failure does not independently authorize `core/refund`.
* Absence of `swap/fiat_confirmed` is not proof that fiat was not received.
* A timeout does not select a default winner unless the pinned profile explicitly defines the relevant authorization and recovery behavior.
* A lost RPC response is not proof that an operation failed.
* A daemon restart does not authorize replaying an economic operation.
* An ambiguous payment state must be reconciled before final publication.
* A fork or invalid action chain must freeze unsafe economic progression.

If the HTLC expires and Lightning confirms the payment failed, the service records that fact and applies the valid recovery policy. It must not pretend that a refund authorization occurred if none was published.

If a payment outcome cannot be determined, mark the execution for reconciliation and avoid publishing a false final state.

---

## 13. Persistent storage and idempotency

The service must be resilient to crashes between authorization, execution, and publication.

Persist at least:

### Swap record

* Swap ID and coordination root event ID.
* Customer and agent identities.
* Bound escrow authority and resolver.
* Exact descriptor event ID and coordinate.
* Pinned protocol and profile versions.
* Amount and currency terms.
* Funding invoice reference and payment hash.
* Encrypted preimage, where required.
* Actual accepted HTLC expiry information.
* Current validated action-chain tip.
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
* Incoming payment reference.
* Outgoing payment reference.
* Attempt count and status.
* Last reconciled node state.
* Last error and recovery checkpoint.
* Final verified outcome, if any.

Use transactions, compare-and-swap updates or equivalent concurrency controls to prevent competing workers from executing incompatible operations.

Before retrying any Lightning operation, reconcile the actual node state. Never infer that an operation failed solely because a request timed out.

The database is an execution journal. It does not override the validated public action chain or the actual Lightning state.

---

## 14. Service API

The exact API must be defined in the versioned OpenAPI or AsyncAPI artifact referenced by the PIP-01 descriptor.

A service may expose operations equivalent to:

| Operation                | Purpose                                                              |
| ------------------------ | -------------------------------------------------------------------- |
| Create escrow            | Create a swap-bound escrow instance.                                 |
| Get escrow status        | Retrieve the reconciled escrow state.                                |
| Get funding instructions | Deliver a private funding invoice or supported alternative.          |
| Submit payout invoice    | Register or replace the agent's payout invoice where permitted.      |
| Submit evidence          | Submit authenticated evidence through the supported private channel. |
| Get execution status     | Retrieve the reconciled incoming and outgoing payment states.        |
| Request reconciliation   | Trigger an authenticated reconciliation of external payment state.   |

The API must define authentication, authorization, error responses, idempotency, versioning, rate limits and privacy requirements.

**Never expose a public endpoint that accepts an arbitrary payment hash and preimage or executes a raw settlement command.**

The execution worker must consume a durable, validated directive. Internal node operations must be isolated from public API processes.

---

## 15. Testing

### 15.1 Protocol tests

* Valid and invalid root signatures.
* Invalid descriptor references.
* Unsupported protocol and profile versions.
* Unauthorized actors.
* Invalid predecessors and replayed actions.
* Conflicting forks.
* Duplicate authorizations.
* Invalid resolver assignments.
* Settlement without prior authorization.
* Refund without prior authorization or a valid resolution effect.
* Timeout with no valid economic authorization.
* Snapshot disagreement with immutable history, where snapshots are supported.

### 15.2 Lightning feasibility tests

On regtest, prove:

* The selected LDK stack supports the required hold-invoice construction.
* A funding payment can remain pending as required.
* The settlement secret matches the expected payment hash.
* Claim and failure operations work as expected.
* Payment expiry returns the expected external result.
* Payment events and node state survive process restarts.
* Unsupported payment modes fail safely.
* Partial or multipart funding is handled according to the declared policy.

### 15.3 Execution tests

* Normal release after valid authorization.
* Refund after valid authorization.
* Unauthorized settlement attempt.
* Unauthorized refund attempt.
* Duplicate execution directive.
* Incoming settlement succeeds but outgoing payout remains pending.
* Outgoing payout fails and is safely retried.
* RPC timeout with an unknown outcome.
* Crash after persisting intent but before executing.
* Crash after execution but before publishing the final action.
* HTLC expiry during a dispute.
* Dispute opened too late for safe resolution.
* External payment failure without refund authorization.
* Recovery after the daemon restarts.
* Fork discovered while an execution is pending.

### 15.4 End-to-end environment

Use:

* Bitcoin Core regtest.
* The selected LDK node implementation.
* Customer and agent test wallets.
* A local Nostr relay.
* Separate test identities for the customer, agent, escrow authority, and resolver.
* A persistent database.
* Automated assertions against both the public action chain and the actual Lightning payment state.

A successful test must verify both protocol correctness and the external payment result. A final Nostr event alone is not proof that the funds moved as intended.

---

## 16. Security and deployment

### Key and secret management

* Encrypt Nostr signing keys at rest.
* Encrypt the preimage where the selected construction requires the service to retain it.
* Separate resolver signing credentials from Lightning execution credentials.
* Keep node credentials out of the public API and general application workers.
* Redact invoices, preimages, private evidence and credentials from logs.
* Limit access to production secrets and audit sensitive operations.

### Operational monitoring

Alert on:

* Pending HTLCs approaching their safety deadline.
* Disputes approaching their resolution deadline.
* Incoming settlement succeeded but payout remains pending.
* Reconciliation failures.
* Repeated or conflicting authorization attempts.
* Nostr relay disconnections.
* Database or secret-store failures.
* Unexpected payment state transitions.
* A mismatch between the public coordination chain and external payment state.

### Operator accountability

Signed actions and durable execution records make it possible to audit intended policy and reported outcomes. They do not prevent a malicious or compromised operator from using its Lightning credentials outside the protocol.

### Production readiness

Do not deploy with real-value trades until the implementation has:

* Demonstrated the required hold-invoice behavior on regtest.
* Validated the exact PIP and profile revisions it supports.
* Published a complete service schema.
* Passed authorization, failure, restart and reconciliation tests.
* Undergone independent review of the Lightning construction and secret handling.
* Documented the operator trust model and recovery limitations.

---

## 17. Future direction: stronger settlement enforcement

The current design is operator-controlled. A future construction may investigate whether a supported and independently reviewed cryptographic payment mechanism can reduce unilateral settlement control.

A PTLC or adaptor-signature proposal must define and prove:

* Who can settle the payment.
* Which secret or signature enables settlement.
* What happens when the resolver disappears.
* Whether either trading participant can settle unilaterally.
* How disputes freeze or constrain economic progression.
* How timeout and recovery operate.
* Whether the required construction is supported by the selected Lightning implementation.

Do not advertise a PTLC/adaptor-signature design as cryptographically enforced merely because it includes an adaptor signature or resolver key. The complete construction must establish the claimed security properties.

Any future mechanism should advertise its distinct custody, authorization, timeout and recovery guarantees through a separate PIP-01 subtype and versioned service schema.

---

## 18. Implementation milestones

1. **LDK feasibility:** Demonstrate the required held-payment lifecycle on regtest.
2. **Lightning adapter:** Build a narrow interface for payment observation, claim/failure, payout tracking and reconciliation.
3. **PIP-02 validator:** Validate signatures, root binding, linked actions, authority and supported versions.
4. **PIP-01 service schema:** Publish the concrete service API and authorization contract.
5. **Durable execution journal:** Persist intent before operations and reconcile after restarts.
6. **Authorized release/refund:** Enforce authorization and verify external outcomes.
7. **Dispute integration:** Validate resolver effects and execute only permitted outcomes.
8. **Recovery testing:** Cover expiry, ambiguous results, crashes, forks and partial settlement.
9. **Interoperability tests:** Publish shared test vectors for supported protocol and profile versions.
10. **Independent review:** Review the Lightning construction, authorization boundaries, secret management and recovery behavior.

## License

MIT
