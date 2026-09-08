# Pontmore HTLC Escrow — Implementation Guide

A step-by-step guide to building a Pontmore-compatible, Nostr-coordinated, node-controlled Lightning hold-invoice escrow. Under PIP-01 this is `custodial_escrow` with a Lightning backend because the operator controls the invoice claim and outgoing payout.

**Normal path:** The customer funds the escrow node's hold invoice; after release authorization, the node settles it and pays the agent's separate payout invoice.
**Dispute path:** Either party raises a dispute. An authorized solver selects release or refund, and the escrow node executes that outcome through LND.

The canonical [Pontmore protocol repository](https://github.com/pontmore/protocol) is the source of truth. PIP-01 defines descriptor-level compatibility, PIP-02 defines the public event grammar, and PIP-03 defines the dispute and timeout policy boundary. Hold-invoice RPCs, authentication, payloads, errors, state names, and authorization rules are implementation behavior that MUST be defined in the OpenAPI or AsyncAPI document referenced by the PIP-01 descriptor.

Canonical role mapping:

| Protocol role | UI alias | Responsibility |
|---------------|----------|----------------|
| `agent` | seller | Provides the traded claim and a payout invoice |
| `customer` | buyer | Requests the swap and funds the hold invoice |
| escrow operator | node | Creates, monitors, settles, or cancels the hold invoice |
| solver | arbiter | Reviews evidence and signs a release/refund directive |

Public fields such as `agent`, `customer`, and `actor_role` use the protocol names. Buyer and seller are only explanatory aliases in this guide.

---

## 1. Environment setup

### Dependencies

- Node.js ≥ 20 or Rust ≥ 1.75
- **Escrow operator** must run LND with hold-invoice, payment, and reconciliation support
- **Customer** needs a Lightning wallet capable of paying invoices
- **Agent** needs a wallet capable of creating a payout invoice
- **Solver** needs a Nostr key and no direct Lightning credentials
- Access to Nostr relays (at least one writable relay)

### Role requirements

| Role    | Lightning node | Hold-invoice | HTLC monitoring | Nostr keypair |
|---------|---------------|--------------|-----------------|---------------|
| Agent   | wallet only   | payout only  | n/a             | required      |
| Customer| wallet only   | n/a          | n/a             | required      |
| Operator| required      | must create  | must monitor    | required      |
| Solver  | none required | n/a          | n/a             | required      |

The escrow node owns the hold invoice and preimage and therefore enforces outcomes against both trading parties. Solver keys authorize policy decisions but have no LND credentials. This separation limits solver compromise, but the node operator remains trusted because it can access both the preimage and payment RPCs.

---

## 2. Nostr identity and relay layer

### 2.1 Keypair management

Each customer, agent, operator, and solver generates or loads a Nostr keypair appropriate to its role.

**Agent nsec** is used for:
- Signing PIP-02 transition, evidence, and note events
- NIP-44 encryption for submitting payout instructions and private evidence
- Gift Wrap encapsulation for private payloads

**Customer nsec** is used for:
- Signing the PIP-02 swap request and relevant transition, evidence, and dispute events
- NIP-44 decryption of the Gift Wrap invoice
- Signing dispute events if needed

**Escrow operator nsec** is used for:
- Signing the k30361 escrow descriptor
- Signing execution transitions, evidence, and snapshots

**Solver nsec** is used for:
- Taking an assigned dispute
- Signing release or refund authorization transitions
- Signing minimal public resolution notes
- NIP-44 decryption of Gift Wrap evidence payloads

Never expose nsec in plaintext on disk.

### 2.2 Relay subscriptions

Each participant maintains persistent WebSocket connections to at least one writable relay.

**Agent subscribes to:**
- Swap request events (`kind 7300`) addressed to the agent
- Transition, evidence, dispute, and note events (`7301` through `7304`) correlated with active swap IDs
- Gift Wrap events addressed to the agent's npub

**Customer subscribes to:**
- Transition, evidence, dispute, and note events (`7301` through `7304`) correlated with the swap ID
- Optional snapshots (`kind 30362`) for fast lookup, verified against immutable history
- Gift Wrap events addressed to the customer's npub, including the BOLT 11 invoice

**Escrow operator subscribes to:**
- Swap requests (`kind 7300`) that reference its descriptor
- Disputes (`kind 7303`) addressed to its npub and related transition/evidence history
- Gift Wrap events addressed to its npub, including sensitive evidence payloads

### 2.3 NIP-44 encryption

Private payloads use a maintained NIP-44 implementation from the selected Nostr library. Implementations MUST NOT replace NIP-44 with the simplified ECDH/encryption sketch previously shown here; key derivation, padding, nonce handling, versioning, and authentication must follow the NIP exactly.

### 2.4 Gift Wrap (private execution lane)

Private payloads — BOLT 11 invoice strings, delivery proofs, dispute evidence — travel through the Gift Wrap lane. These are NIP-44 encrypted messages sealed inside wrapper events.

Public PIP-02 events carry the request and append-only lifecycle history. The private lane transports non-public payloads and is supplementary: it never overrides the public request, transition, evidence, dispute, note, or snapshot history.

---

## 3. Swap request protocol

### 3.0 Escrow descriptor prerequisite

Before accepting requests, the escrow operator publishes an addressable PIP-01 descriptor (`kind 30361`) with a stable `d` tag. A minimum descriptor for this implementation is:

```json
{
  "kind": 30361,
  "tags": [
    ["d", "lightning-custodial-escrow-v1"],
    ["network", "lightning"]
  ],
  "content": "{\"version\":1,\"escrow_type\":\"custodial_escrow\",\"networks\":[\"lightning\"],\"funding_rules\":{\"funding_threshold\":1,\"participant_count\":1},\"dispute_rules\":{\"policy\":\"pip03\"},\"reference_format\":\"bolt11_or_custodial_escrow_reference\",\"service\":{\"schema\":{\"type\":\"openapi\",\"url\":\"https://escrow.example.com/pontmore-lightning-custodial-v1.openapi.json\"}},\"updated_at\":1724000000}",
  "created_at": 1724000000
}
```

The repeated `network` tag is a discovery index; `content.networks` is canonical. `funding_threshold: 1` and `participant_count: 1` mean one declared participant must fund the escrow. They do not grant settlement authority. The schema URL must use HTTPS, avoid private or unsafe destinations, and should identify an immutable or versioned artifact. Clients must apply bounded fetches, redirect limits, content-type checks, and response-size limits before trusting it.

### 3.1 Request structure

The root is an immutable PIP-02 swap request event (`kind 7300`). Its content includes all required canonical fields:

```json
{
  "kind": 7300,
  "tags": [
    ["p", "<agent_npub>"],
    ["a", "30361:<operator_npub>:<descriptor_d_tag>"]
  ],
  "content": "{\"version\":1,\"swap_id\":\"...\",\"swap_type\":\"...\",\"agent\":\"...\",\"customer\":\"...\",\"escrow_reference\":\"...\",\"fiat\":{...},\"bitcoin\":{...},\"expiry\":1724000000}",
  "created_at": 1724000000
}
```

`escrow_reference` binds the request to the selected kind `30361` descriptor or escrow claim using the descriptor's declared `reference_format`. The exact `fiat`, `bitcoin`, and private execution payloads must follow the selected implementation's service schema. Raw invoices, preimages, wallet identifiers, and internal credentials stay out of public events.

### 3.2 Request and acceptance flow

1. Customer and agent negotiate terms through a declared channel.
2. Customer creates and publishes the immutable `kind 7300` request.
3. Agent validates the referenced current agent definition (`kind 30360`), escrow descriptor (`kind 30361`), and service schema.
4. Agent accepts or rejects with a `kind 7301` transition containing `swap_id`, `state`, `prev_state`, `actor_role`, `reason`, and `created_at`.
5. Escrow operator observes the request and transition log for possible dispute tracking.

A normal Nostr event has one author and one signature. This implementation therefore MUST NOT describe a request as "doubly signed." Multi-party consent is represented by separate signed, linked events in the append-only history.

---

## 4. Node-controlled Lightning escrow

### 4.1 Hold invoice creation (escrow node)

The escrow node creates and controls the incoming hold invoice. Neither customer nor agent receives its preimage.

1. Generate `p ← CSPRNG(32 bytes)` — the payment preimage.
2. Compute `H = SHA256(p)`.
3. Create a hold invoice with `hash = H`, the requested amount, an invoice expiry, and a final CLTV delta that satisfy the referenced service schema. Record the actual accepted HTLC expiry height separately when payment arrives.

**LND (gRPC):**
```
lncli addholdinvoice --hash=<hex(H)> --amt=<amount_msat>m --memo="Pontmore swap <id>" --cltv_expiry=<htlc_expiry_blocks>
```

**Core Lightning:** No standard RPC equivalent to LND's `addholdinvoice` is assumed by this guide. A deployment claiming CLN support MUST provide and test a dedicated hold-invoice plugin, document its `htlc_accepted` handling in the referenced service schema, and ensure it safely handles MPP, retries, restart recovery, cancellation, and expiry. The standard `invoice` RPC is not a substitute.

4. Encrypt and durably store `swap_id → (payment_hash, preimage, bolt11_invoice, amount_msat, state)`.
5. Deliver the BOLT 11 invoice string to the customer through the private lane.
6. Publish an `INVOICED` `kind 7301` transition.
7. Begin monitoring for HTLC arrival.

The preimage `p` never leaves the escrow execution service. It is not available to trading parties, solvers, logs, public events, or general application workers.

### 4.2 HTLC monitoring (escrow node)

**LND:** Subscribe to `SubscribeInvoices` gRPC stream. Watch for `state = ACCEPTED`.

**CLN:** Use the deployment's declared and tested hold-invoice plugin; do not defer arbitrary HTLCs with an incomplete hook implementation.

On HTLC arrival:
1. Match the `payment_hash` to a swap ID.
2. After the complete expected payment is accepted, publish a `FUNDED` `kind 7301` transition. For MPP, no partial shard set is `FUNDED`.
3. Optionally publish a reference-style funding proof as `kind 7302`; do not expose a raw invoice or private routing data.
4. Begin normal-path evaluation or wait for dispute.

### 4.3 Agent payout instruction

Before release, the agent sends a BOLT 11 payout invoice through the private lane. The node validates:

- destination and network policy
- exact expected payout amount after declared fees
- invoice expiry and remaining execution window
- uniqueness and binding to `swap_id`
- payment hash has not been used for another payout

The raw payout invoice remains private. A public `kind 7302` event may contain only an opaque reference or hash.

### 4.4 Normal path — release (escrow node)

When the delivery condition is met (delivery confirmed by buyer or external oracle):

1. Validate the required release authorization and atomically persist an idempotent execution record.
2. Call `settle(preimage)` on the incoming hold invoice.
3. Mark the incoming payment `SETTLED` after LND reconciliation.
4. Pay the agent's validated payout invoice.
5. Publish `PAYOUT_PENDING` while payment is unresolved and `SETTLED` only after payout succeeds.
6. Retry safely or request a replacement invoice according to the service schema.

### 4.5 Normal path — refund (escrow node)

When the refund condition is met (timeout, mutual agreement):

1. Validate refund authorization or the declared timeout fallback.
2. Call `cancelinvoice` on the incoming hold invoice.
3. Reconcile the confirmed Lightning result and publish a `REFUNDED` `kind 7301` transition.
4. HTLC funds return to the customer.

### 4.6 Hold vs standard invoices

Hold invoices are required. A standard invoice auto-settles on HTLC arrival, removing the node's ability to refund. With a hold invoice:
- The customer's HTLC arrives at the escrow node.
- Only the escrow execution service can `settle(preimage)` or `cancelinvoice`.
- The node must execute before the HTLC's CLTV expiry; after expiry the HTLC auto-fails and funds return to the customer.
- The dispute window must fit within the CLTV window.

---

## 5. Swap state machine (PIP-02)

### 5.1 States

```
REQUESTED           → immutable kind 7300 request exists
ACCEPTED            → agent accepted the request
INVOICED            → hold invoice delivered privately
FUNDED              → complete HTLC payment accepted by the escrow node
DISPUTED            → kind 7303 dispute opened; arbiter reviewing
RELEASE_AUTHORIZED  → release authorized under normal or dispute policy
REFUND_AUTHORIZED   → refund authorized under timeout or dispute policy
PAYOUT_PENDING      → incoming HTLC settled; agent payout not yet confirmed
SETTLED             → incoming settlement and agent payout confirmed
REFUNDED            → Lightning cancellation or expiry confirmed
EXPIRED             → request expired before funding
```

### 5.2 State transitions

```
REQUESTED ──→ ACCEPTED ──→ INVOICED ──→ FUNDED
   │                                      ├──→ RELEASE_AUTHORIZED ──→ PAYOUT_PENDING ──→ SETTLED
   ▼                                      ├──→ REFUND_AUTHORIZED  ──→ REFUNDED
EXPIRED                                   └──→ DISPUTED
                                                    ├──→ RELEASE_AUTHORIZED ──→ PAYOUT_PENDING ──→ SETTLED
                                                    └──→ REFUND_AUTHORIZED  ──→ REFUNDED
```

Each state change is an immutable PIP-02 transition event (`kind 7301`) with coherent `state`, `prev_state`, and `actor_role`. The request (`7300`), transition log (`7301`), evidence references (`7302`), dispute (`7303`), and notes (`7304`) are append-only. A replaceable snapshot (`30362`) is only a fast lookup optimization; immutable history is authoritative.

These state names are this subtype's service behavior, not canonical PIP-02 states. They MUST be enumerated in the referenced service schema together with permitted actors and transitions.

### 5.3 HTLC expiry fallback

If the actual accepted HTLC expires before any resolution:
- Lightning auto-fails the HTLC, funds return to the customer.
- Publish a `REFUNDED` transition after reconciling the Lightning result. `EXPIRED` is reserved for a request that expired before funding.
- If a dispute was pending, the operator may publish a `kind 7304` note explaining that the payment timed out, but must not imply that a later resolution moved funds.

---

## 6. Dispute resolution (arbiter side)

### 6.1 Dispute initiation

Either party publishes a PIP-02 dispute event (`kind 7303`) and a coherent transition to `DISPUTED`:

```json
{
  "kind": 7303,
  "tags": [
    ["e", "<swap_request_event_id>", "<relay_hint>"],
    ["p", "<arbiter_npub>"]
  ],
  "content": "{\"version\":1,\"swap_id\":\"...\",\"dispute_class\":\"...\",\"reason\":\"...\"}",
  "created_at": ...
}
```

Any dispute fee and proof format are implementation-specific and must be declared by the referenced service schema. Clients must not infer a keysend fee requirement from PIP-03 itself.

### 6.2 Evidence submission

Both parties submit evidence via Gift Wrap to the arbiter:
- Delivery proofs, screenshots, tracking information
- Signed messages, timestamps
- Any mutually agreed oracle attestations

Evidence is encrypted to the arbiter's npub and delivered through the Gift Wrap lane.

### 6.3 Arbiter review

The arbiter:
1. Receives the `DISPUTED` event.
2. Collects evidence from both parties via Gift Wrap.
3. Evaluates the request, append-only history, private terms, and evidence under the declared policy.
4. Publishes a signed `kind 7301` authorization transition to `RELEASE_AUTHORIZED` or `REFUND_AUTHORIZED`.
5. Optionally publishes a `kind 7304` public note with the minimum reasoning necessary.

```json
{
  "kind": 7301,
  "tags": [
    ["e", "<swap_request_event_id>", "<relay_hint>"],
    ["e", "<dispute_event_id>", "<relay_hint>"],
    ["p", "<buyer_npub>"],
    ["p", "<seller_npub>"]
  ],
  "content": "{\"swap_id\":\"...\",\"state\":\"RELEASE_AUTHORIZED\",\"prev_state\":\"DISPUTED\",\"actor_role\":\"escrow_operator\",\"reason\":\"customer_claim_confirmed\",\"created_at\":1724000000}",
  "created_at": 1724000000
}
```

The arbiter signs the Nostr event with their nsec. The normal event signature authenticates the transition; do not duplicate a signature inside `content`.

### 6.4 Decision enforcement

The escrow service verifies the solver assignment, write permission, event signature, current state, and idempotency key before execution:

- **RELEASE_AUTHORIZED** → node settles the incoming hold invoice, enters `PAYOUT_PENDING`, pays the agent invoice, and publishes `SETTLED` after reconciliation
- **REFUND_AUTHORIZED** → node cancels the incoming hold invoice and publishes `REFUNDED` after reconciliation

Neither party can override the decision because neither party has the preimage or LND credentials. The operator can still violate policy or fail, so this is operator-controlled enforcement rather than trustless three-party cryptography.

### 6.5 Dispute window enforcement

The solver and escrow node MUST resolve and execute before the deadline derived from the actual accepted HTLC expiry height. The implementation should:
1. Track the request and dispute deadlines from the service schema and the accepted HTLC expiry height observed by the escrow execution service.
2. Set an internal block-height deadline that preserves the declared execution and safety buffers.
3. Bind the timeout to the explicit non-`mutual_consent` fallback declared by the descriptor or service schema, as PIP-03 requires.
4. If the payment expires, reconcile it as `REFUNDED` and publish the corresponding transition.
5. Alert both parties that the HTLC is approaching expiry.

---

## 7. Escrow node and solver service

### 7.1 Core loop

```
while running:
    ensure k30361 descriptor and service schema are published and up-to-date
    accept PIP-02 request, transition, evidence, dispute, and note events
    for each new swap request referencing this descriptor:
        record swap for potential dispute tracking
    for each dispute event:
        notify arbiter operator
        collect evidence from both parties
        evaluate against swap conditions
        if decision reached:
            solver publishes signed RELEASE_AUTHORIZED or REFUND_AUTHORIZED transition
            execution service validates and executes the directive through LND
        if deadline approaching without decision:
            execute the declared PIP-03 fallback and publish its transition
```

The solver does **not** generate invoices, hold preimages, receive LND credentials, or call settlement RPCs. The escrow execution service performs those operations. Production deployments should isolate solver authorization from Lightning execution and require an authenticated, auditable directive between them.

### 7.2 Storage schema

```
swaps:
  id                  TEXT PRIMARY KEY
  buyer_npub          TEXT NOT NULL
  seller_npub         TEXT NOT NULL
  amount_msat         INTEGER NOT NULL
  payment_hash        TEXT                  # SHA256(preimage), bound by private execution data
  preimage_ciphertext BLOB                  # encrypted; execution service only
  payout_invoice_hash TEXT                  # private invoice commitment
  invoice_expires_at  INTEGER               # BOLT 11 invoice expiry time
  htlc_expiry_height  INTEGER               # actual accepted HTLC expiry height
  dispute_deadline    INTEGER               # policy deadline
  state               TEXT NOT NULL         # subtype state declared by the service schema
  request_json        TEXT                  # immutable kind 7300 request
  created_at          INTEGER NOT NULL
  updated_at          INTEGER NOT NULL

disputes:
  id                  TEXT PRIMARY KEY
  swap_id             TEXT REFERENCES swaps(id)
  raised_by_npub      TEXT NOT NULL
  reason              TEXT
  evidence            TEXT                  # JSON array of evidence references
  decision            TEXT                  # RELEASE_AUTHORIZED | REFUND_AUTHORIZED
  decision_event_id   TEXT                  # Nostr event ID of resolution transition
  created_at          INTEGER NOT NULL
  resolved_at         INTEGER

executions:
  directive_event_id  TEXT PRIMARY KEY
  swap_id             TEXT REFERENCES swaps(id)
  operation           TEXT NOT NULL         # RELEASE | REFUND | PAYOUT
  status              TEXT NOT NULL         # PENDING | SUBMITTED | CONFIRMED | FAILED
  attempt_count       INTEGER NOT NULL
  lightning_reference TEXT
  last_error          TEXT
  created_at          INTEGER NOT NULL
  updated_at          INTEGER NOT NULL
```

### 7.3 Service endpoints

| Method | Path                       | Description                         |
|--------|----------------------------|-------------------------------------|
| GET    | `/swap/:id`                | Get swap state                      |
| POST   | `/swap/:id/dispute`        | Raise a dispute as either party     |
| POST   | `/swap/:id/evidence`       | Submit authenticated evidence       |
| GET    | `/swap/:id/decision`       | Get solver decision                 |
| POST   | `/swap/:id/payout-invoice` | Submit/replace agent payout invoice |
| GET    | `/swap/:id/execution`      | Get reconciled Lightning execution  |

LND execution is not exposed as a general public endpoint. The execution worker consumes a durable directive only after validating the assigned solver, permission level, signature, expected `prev_state`, deadline, and idempotency key. If an internal `/execute` operation exists, it must be authenticated service-to-service and unavailable from the public listener.

---

## 8. Client integration

### 8.1 Buyer flow

1. Load Nostr keypair.
2. Connect to relays and subscribe to the PIP-02 event kinds.
3. Negotiate terms with seller.
4. Publish the immutable kind 7300 request and wait for the agent's acceptance transition.
5. Receive BOLT 11 hold invoice via Gift Wrap.
6. Pay the invoice via Lightning.
7. Wait for the `FUNDED` transition and validate its append-only history.
8. Confirm delivery (if goods received).
9. Wait for the `SETTLED` transition and corresponding Lightning reconciliation result.

If delivery never arrives:
1. Publish `DISPUTED` event before dispute window closes.
2. Submit evidence to arbiter via Gift Wrap.
3. Wait for the arbiter's `RELEASE_AUTHORIZED` or `REFUND_AUTHORIZED` transition.
4. If refund is authorized, wait for the escrow node's reconciled `REFUNDED` transition.

### 8.2 Agent flow

1. Load the agent's Nostr keypair and Lightning wallet.
2. Connect to relays.
3. Validate the customer's request and publish an acceptance transition.
4. Create a payout invoice for the exact expected amount and deliver it privately to the escrow node.
5. Wait for the node's `INVOICED` and `FUNDED` transitions.
6. Fulfil the traded obligation.
7. Participate in normal release authorization or submit evidence during a dispute.
8. Verify receipt of the outgoing payout; the agent never receives the escrow preimage.

### 8.3 Escrow operator and solver setup

1. Run LND with narrowly scoped invoice and payment credentials in the execution service.
2. Store preimages encrypted under a vault or HSM-backed data key.
3. Load separate operator and solver signing keys.
4. Connect to relays and subscribe to requests and dispute history.
5. Reconcile all nonterminal executions with LND after every restart.

---

## 9. Testing

### 9.1 Unit tests

- Swap request signing and validation (wrong author, expiry, missing canonical fields, invalid escrow reference).
- Transition authorization and `prev_state` coherence.
- Hold-invoice creation (verify payment_hash matches SHA256(preimage)).
- State machine transitions — reject invalid transitions (e.g., SETTLED → DISPUTED).
- Dispute event signing and validation.
- Arbiter resolution-transition signing, actor authorization, and signature verification.
- NIP-44 encryption round-trip.
- Gift Wrap encapsulation and decapsulation.

### 9.2 Integration tests

- **Regtest Lightning network.** Use Polar with an escrow LND node plus customer and agent wallets.
- **Local Nostr relay.** nostr-rs-relay in Docker.
- **Normal release path.** Request → acceptance → node hold invoice → customer HTLC → FUNDED → RELEASE_AUTHORIZED → node settles → PAYOUT_PENDING → node pays agent → SETTLED.
- **Normal refund path.** Request → node hold invoice → customer HTLC → FUNDED → REFUND_AUTHORIZED → node cancels → REFUNDED.
- **Dispute release.** FUNDED → kind 7303 dispute → DISPUTED → authorized solver release → node settles and pays agent.
- **Dispute refund.** FUNDED → kind 7303 dispute → DISPUTED → authorized solver refund → node cancels the HTLC.
- **Unauthorized solver.** Submit a validly signed resolution from an unassigned or read-only solver and verify no LND operation occurs.
- **Payout failure after settlement.** Settle the incoming HTLC, fail the outgoing payment, verify `PAYOUT_PENDING`, durable retries, and no duplicate payment.
- **HTLC expiry before dispute resolution.** Request → hold invoice with insufficient window → FUNDED → dispute raised too late → HTLC expires → REFUNDED.
- **Conflicting events.** Publish transitions with incoherent `prev_state` values and verify clients reject them when materializing the append-only history.
- **Snapshot disagreement.** Publish a stale or incorrect kind 30362 snapshot and verify clients prefer immutable request and transition history.

### 9.3 Test harness

A scripted runner that:
1. Spawns an escrow LND node and customer/agent regtest wallets.
2. Starts a local Nostr relay.
3. Creates keypairs for customer, agent, operator, and solver.
4. Operator publishes the k30361 descriptor and service schema.
5. Executes each scenario and asserts: state transitions, HTLC settle/cancel outcomes, arbiter decision signatures.

---

## 10. Deployment considerations

### 10.1 Security

- **nsec storage.** All keypairs encrypted at rest. Arbiters should use vaulted keys.
- **Node preimage storage.** `preimage_p` encrypted at rest with a vault- or HSM-protected data encryption key.
- **Lightning execution access.** The isolated execution service needs narrowly scoped invoice read/write and outgoing payment permissions. Solver and general API processes receive no macaroon.
- **Payout safety.** Use durable idempotency, outgoing payment lookup by payment hash, compare-and-swap state changes, and restart reconciliation before retries.
- **Dispute fee.** A small Lightning payment (keysend) should accompany dispute events to deter abuse.

### 10.2 Operational

- **Deadline separation.** Keep request expiry, BOLT 11 invoice expiry, final CLTV delta, actual accepted HTLC expiry height, dispute deadline, and execution buffer as distinct values. Do not compare seconds directly with block deltas.
- **Arbiter availability.** The arbiter must be online and responsive within the dispute window. SLAs should be defined.
- **Monitoring.** Alert on: FUNDED swaps approaching dispute window end without resolution, arbiter decision deadline approaching, relay disconnections.
- **Operator accountability.** Signed solver directives and node execution evidence make policy violations auditable, but cannot prevent a compromised operator from misusing its LND access.

### 10.3 Trust model and future PTLC upgrade

The node-controlled design enforces outcomes against both trading parties, but the operator remains trusted. A future non-custodial construction would require a separately specified and reviewed PTLC/adaptor-signature subtype:

- Define a separate descriptor and service schema for a reviewed PTLC construction that requires the arbiter's adaptor signature.
- The seller cannot settle unilaterally after a dispute is raised — the arbiter's adaptor is required.
- The arbiter provides the adaptor to the winning party.
- This is `settlement_enforcement: cryptographic`.

The PIP-02 event grammar and PIP-03 policy boundary may remain compatible, but custody, authorization, timeout, and settlement semantics must be accurately advertised through PIP-01 and the referenced service schema.
