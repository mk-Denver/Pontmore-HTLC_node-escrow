# Pontmore Lightning Hold-Invoice Escrow — Implementation Guide (Alternative)

**Non-custodial, agent-owned preimage, P2P Lightning HTLC with arbiter dispute-only fallback.**

This is the **alternative** implementation guide for the canonical PIP-01 `lightning_hold_invoice` escrow subtype. Unlike the primary `custodial_escrow` guide in `IMPL_GUIDE.md`, this construction places the hold invoice and its preimage under the **agent's** control. The escrow operator takes no custody of funds, holds no preimage, and cannot settle or cancel the HTLC. Dispute resolution is cooperative: the arbiter publishes signed Nostr decisions, and the agent is expected to honor them by calling `settle` or `cancelinvoice` on its own node.

Use this subtype only when the agent is trusted to honor arbiter decisions or when off-chain reputation, slashing, or future PTLC enforcement makes defection unprofitable. For operator-enforced outcomes, use the `custodial_escrow` construction instead.

---

## When to use this subtype

| Property | `lightning_hold_invoice` (this guide) | `custodial_escrow` (primary guide) |
|----------|--------------------------------------|------------------------------------|
| Preimage owner | Agent | Operator |
| Hold-invoice node | Agent's LND/CLN | Operator's LND |
| Operator custody | None | Full |
| Dispute enforcement | Cooperative only | Cryptographic against both parties |
| Agent defection risk | Yes — agent can ignore arbiter | No |
| Operator trust required | No | Yes |
| Failure mode | Agent keeps funds until CLTV | Operator compromise/insolvency |

This subtype is appropriate when:

- the agent is a long-lived, reputationally-bound participant
- the dispute window fits comfortably inside the HTLC CLTV window
- off-chain accountability (marketplace bans, slashing, attestations) is sufficient deterrence
- the deployment cannot or will not run an operator LND node with custody

---

## Protocol Conformance

This implementation conforms to:

- [PIP-00](https://github.com/pontmore/protocol/blob/main/PIP-00-agent-definition.md) — agent definition and discovery
- [PIP-01](https://github.com/pontmore/protocol/blob/main/PIP-01-escrow-descriptor.md) — escrow descriptor, with `escrow_type: "lightning_hold_invoice"`
- [PIP-02](https://github.com/pontmore/protocol/blob/main/PIP-02-swap-state-machine.md) — public swap event lifecycle
- [PIP-03](https://github.com/pontmore/protocol/blob/main/PIP-03-dispute-policy.md) — dispute and timeout policy

The canonical [Pontmore protocol repository](https://github.com/pontmore/protocol) is the source of truth. This guide uses PIP-01 descriptors and PIP-02 events but does not restate them.

---

## Roles

| Protocol role | UI alias | Responsibility |
|---------------|----------|----------------|
| `agent` | seller | Owns the hold invoice and preimage; settles or cancels on their own node |
| `customer` | buyer | Requests the swap, pays the hold invoice, confirms delivery |
| `arbiter` (solver) | arbiter | Reviews disputes and publishes signed release/refund decisions on Nostr |
| escrow operator | — | **Not used.** No custody, no LND credentials, no preimage access |

The arbiter does **not** run a Lightning node, hold the preimage, monitor HTLCs, or call settlement RPCs. It only signs and publishes Nostr decisions.

---

## Escrow descriptor

The agent publishes an addressable PIP-01 descriptor advertising the `lightning_hold_invoice` subtype:

```json
{
  "kind": 30361,
  "tags": [
    ["d", "lightning-hold-invoice-v1"],
    ["network", "lightning"],
    ["p", "<agent_npub>"]
  ],
  "content": "{\"version\":1,\"escrow_type\":\"lightning_hold_invoice\",\"networks\":[\"lightning\"],\"funding_rules\":{\"funding_threshold\":1,\"participant_count\":1},\"dispute_rules\":{\"policy\":\"pip03\"},\"reference_format\":\"bolt11\",\"service\":{\"schema\":{\"type\":\"openapi\",\"url\":\"https://agent.example.com/pontmore-lightning-hold-invoice-v1.openapi.json\"}},\"updated_at\":1724000000}",
  "created_at": 1724000000
}
```

Notes:

- `escrow_type` is exactly `lightning_hold_invoice`, matching the canonical PIP-01 subtype.
- `reference_format` is `bolt11` because the escrow reference the customer funds is a Lightning hold invoice produced by the agent.
- The `service` block is optional. If present, the schema URL must use `https`, avoid unsafe destinations, and resolve to an immutable or versioned artifact. Clients must apply bounded fetches, redirect limits, content-type checks, and response-size limits.
- `funding_threshold: 1` and `participant_count: 1` mean one declared participant (the customer) must fund the escrow. They describe funding cardinality, not settlement authority. Settlement authority on this subtype belongs to the agent because the agent owns the preimage.

---

## Swap request

The root is an immutable PIP-02 `kind 7300` request:

```json
{
  "kind": 7300,
  "tags": [
    ["p", "<agent_npub>"],
    ["a", "30361:<agent_npub>:lightning-hold-invoice-v1"]
  ],
  "content": "{\"version\":1,\"swap_id\":\"...\",\"swap_type\":\"...\",\"agent\":\"...\",\"customer\":\"...\",\"escrow_reference\":\"pending\",\"fiat\":{...},\"bitcoin\":{...},\"expiry\":1724000000}",
  "created_at": 1724000000
}
```

`escrow_reference` is initially `pending` because the hold invoice has not yet been created. After the agent creates the invoice, the BOLT 11 string is delivered privately through the Gift Wrap lane; the public `escrow_reference` is then updated only through append-only transition events, never by mutating the original request.

A normal Nostr event has one author and one signature. Multi-party consent is represented by separate signed, linked events in the append-only history, not by a "doubly signed" request.

---

## Swap states

```
REQUESTED  →  immutable kind 7300 request exists
ACCEPTED   →  agent accepted the request
INVOICED   →  hold invoice created on the agent's node and delivered privately
FUNDED     →  customer's HTLC accepted by the agent's node
SETTLED    →  agent settled the HTLC with the preimage
REFUNDED   →  agent cancelled the HTLC
DISPUTED   →  kind 7303 dispute opened; arbiter reviewing
CLOSED     →  arbiter decision published (RELEASE or REFUND)
EXPIRED    →  request expired before funding, or HTLC CLTV expired before resolution
```

State transitions:

```
REQUESTED ──→ ACCEPTED ──→ INVOICED ──→ FUNDED ──→ SETTLED            (normal release, agent signs)
                                              ├──→ REFUNDED           (normal refund, agent signs)
                                              └──→ DISPUTED           (either party signs)
                                                          │
                                                          ▼
                                                       CLOSED            (arbiter signs)
REQUESTED ──→ EXPIRED
FUNDED    ──→ EXPIRED                                            (HTLC CLTV expired)
```

Every state change is an append-only `kind 7301` event with coherent `prev_state` and `actor_role`. Signing authority:

- `ACCEPTED`, `INVOICED`, `FUNDED`, `SETTLED`, `REFUNDED` — signed by the **agent**
- `DISPUTED` — signed by whichever party raises it
- `CLOSED` — signed by the **arbiter**

These state names are this subtype's service behavior, not canonical PIP-02 states. They MUST be enumerated in the referenced service schema together with permitted actors and transitions.

---

## Step-by-step implementation

### 1. Keypair management

Each customer, agent, and arbiter generates or loads a Nostr keypair.

**Agent nsec** is used for:
- Signing the kind 30361 escrow descriptor
- Signing PIP-02 transition events (ACCEPTED, INVOICED, FUNDED, SETTLED, REFUNDED)
- NIP-44 shared-key derivation for delivering the BOLT 11 invoice to the customer

**Customer nsec** is used for:
- Signing the PIP-02 swap request and any dispute events
- NIP-44 decryption of the Gift Wrap invoice

**Arbiter nsec** is used for:
- Signing dispute decision events (CLOSED)
- NIP-44 decryption of Gift Wrap evidence payloads

Never expose nsec in plaintext on disk.

### 2. Relay subscriptions

**Agent subscribes to:**
- Swap request events (`kind 7300`) addressed to the agent
- Dispute events (`kind 7303`) tagged with active swap IDs
- Gift Wrap events addressed to the agent's npub

**Customer subscribes to:**
- Transition events (`kind 7301`) correlated with the swap ID
- Optional snapshots (`kind 30362`) for fast lookup, verified against immutable history
- Gift Wrap events addressed to the customer's npub (contains the BOLT 11 invoice)
- Arbiter decision events tagged with the swap ID

**Arbiter subscribes to:**
- Swap request events tagged with the arbiter's npub (to track active swaps)
- Dispute events tagged with the arbiter's npub
- Gift Wrap events addressed to the arbiter's npub (contains evidence payloads)

### 3. NIP-44 and Gift Wrap

Private payloads — BOLT 11 invoice strings, delivery proofs, dispute evidence — travel through the Gift Wrap lane. These are NIP-44 encrypted messages sealed inside wrapper events.

Use a maintained NIP-44 implementation from the selected Nostr library. Do not replace NIP-44 with a simplified ECDH/encryption sketch; key derivation, padding, nonce handling, versioning, and authentication must follow the NIP exactly.

Public PIP-02 events carry the request and append-only lifecycle history. The private lane is supplementary; it never overrides the public request, transition, evidence, dispute, note, or snapshot history.

### 4. Hold invoice creation (agent)

The agent creates the hold invoice on its own Lightning node. Neither the customer nor the arbiter is involved.

1. Generate `p ← CSPRNG(32 bytes)` — the payment preimage.
2. Compute `H = SHA256(p)`.
3. Create a hold invoice with `hash = H`, the agreed amount, and CLTV expiry matching the swap's declared window.

**LND (gRPC):**
```
lncli addholdinvoice --hash=<hex(H)> --amt=<amount_msat>m --memo="Pontmore swap <id>" --cltv_expiry=<htlc_expiry_blocks>
```

**Core Lightning:** A deployment claiming CLN support MUST provide and test a dedicated hold-invoice plugin, document its `htlc_accepted` handling in the referenced service schema, and ensure it safely handles MPP, retries, restart recovery, cancellation, and expiry. The standard `invoice` RPC is not a substitute.

4. Store the mapping: `swap_id → (payment_hash, preimage, bolt11_invoice, amount_msat, state)` encrypted at rest.
5. Deliver the BOLT 11 invoice string to the customer via Gift Wrap.
6. Publish an `INVOICED` `kind 7301` transition.
7. Begin monitoring for HTLC arrival.

The preimage `p` never leaves the agent's process. The arbiter never sees it. The escrow operator role does not exist in this subtype.

### 5. HTLC monitoring (agent)

**LND:** Subscribe to `SubscribeInvoices` gRPC stream. Watch for `state = ACCEPTED`.

**CLN:** Use the deployment's declared and tested hold-invoice plugin.

On HTLC arrival:
1. Match the `payment_hash` to a swap ID.
2. After the complete expected payment is accepted, publish a `FUNDED` `kind 7301` transition. For MPP, no partial shard set is `FUNDED`.
3. Optionally publish a reference-style funding proof as `kind 7302`; do not expose a raw invoice or private routing data.
4. Begin normal-path evaluation or wait for dispute.

### 6. Normal path — release (agent)

When the delivery condition is met (delivery confirmed by customer or external oracle):

1. Call `settle(preimage)` on the agent's hold invoice.
2. Publish a `SETTLED` `kind 7301` transition.
3. Swap is complete.

### 7. Normal path — refund (agent)

When the refund condition is met (timeout, mutual agreement):

1. Call `cancelinvoice` on the agent's hold invoice.
2. Publish a `REFUNDED` `kind 7301` transition.
3. HTLC funds return to the customer. Swap is complete.

### 8. Hold vs standard invoices

Hold invoices are required. A standard invoice auto-settles on HTLC arrival, removing the agent's ability to refund. With a hold invoice:
- The customer's HTLC arrives at the agent's node.
- Only the agent can `settle(preimage)` to claim or `cancelinvoice` to refund.
- The agent must decide before the HTLC's CLTV expiry; after expiry the HTLC auto-fails and funds return to the customer.
- The dispute window must fit within the CLTV window.

---

## Dispute resolution

### Dispute initiation

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

Any dispute fee and proof format are implementation-specific and must be declared by the referenced service schema.

### Evidence submission

Evidence is encrypted to the arbiter's npub using NIP-44 and delivered through the Gift Wrap lane.

### Arbiter review and decision

The arbiter:
1. Receives the `DISPUTED` event.
2. Collects evidence from both parties via Gift Wrap.
3. Evaluates the request, append-only history, private terms, and evidence under the declared policy.
4. Publishes a signed `CLOSED` decision event:

```json
{
  "kind": 7301,
  "tags": [
    ["e", "<swap_request_event_id>", "<relay_hint>"],
    ["e", "<dispute_event_id>", "<relay_hint>"],
    ["p", "<customer_npub>"],
    ["p", "<agent_npub>"]
  ],
  "content": "{\"swap_id\":\"...\",\"state\":\"CLOSED\",\"prev_state\":\"DISPUTED\",\"actor_role\":\"arbiter\",\"decision\":\"RELEASE\",\"reasoning\":\"...\",\"created_at\":1724000000}",
  "created_at": 1724000000
}
```

The arbiter signs the Nostr event with their nsec. The normal event signature authenticates the decision; do not duplicate a signature inside `content`.

### Decision enforcement (cooperative)

The agent is expected to honor the decision:
- **RELEASE** → agent calls `settle(preimage)`, publishes `SETTLED`
- **REFUND** → agent calls `cancelinvoice`, publishes `REFUNDED`

If the agent ignores the decision, the customer has a verifiable signed arbiter event as evidence for off-chain dispute resolution (reputation systems, marketplace bans, legal recourse). The arbiter cannot force settlement because the arbiter has no preimage and no LND credentials.

This is the fundamental limitation of the `lightning_hold_invoice` subtype: enforcement is cooperative, not cryptographic.

### Dispute window enforcement

The arbiter MUST publish the decision before the HTLC CLTV expiry. The arbiter's implementation should:
1. Track `htlc_expiry` from the swap request and the actual accepted HTLC expiry height.
2. Set an internal deadline at `htlc_expiry - buffer` (e.g., 10 blocks).
3. If the deadline passes without a decision, publish a `CLOSED` event with `decision: "EXPIRED"`.
4. Alert both parties that the HTLC will auto-fail.
5. Bind the timeout to the explicit non-`mutual_consent` fallback declared by the descriptor or service schema, as PIP-03 requires.

### HTLC expiry fallback

If the actual accepted HTLC expires before any resolution:
- Lightning auto-fails the HTLC, funds return to the customer.
- Publish a `REFUNDED` transition after reconciling the Lightning result. `EXPIRED` is reserved for a request that expired before funding.
- If a dispute was pending, the arbiter may still publish a decision for reputational purposes, but the HTLC is gone.

---

## Security properties

- **Normal-path atomicity.** The customer's funds are locked in the hold-invoice HTLC. The agent can only claim them by settling with `p`. The agent's refund path is `cancelinvoice`. Both are Lightning-enforced.
- **Non-custodial.** No operator holds the preimage, LND credentials, or the ability to settle/cancel. The arbiter has no custody.
- **Dispute auditability.** Arbiter decisions are signed Nostr events with verifiable Schnorr signatures. Any relay observer can verify the arbiter made a particular decision at a particular time.
- **Non-repudiation.** All agreements, state transitions, and arbiter decisions are signed.
- **Agent trust.** The agent must be trusted to honor arbiter decisions. A malicious agent can ignore the decision and keep the HTLC unresolved until CLTV expiry refunds them (or settle and steal).

---

## Limitations

- **Cooperative enforcement.** On dispute, the agent must honor the arbiter's decision. A malicious agent can ignore the decision. Enforcement is social/reputational, not cryptographic.
- **CLTV-bound dispute window.** Dispute resolution must complete before HTLC expiry. Long disputes on short-CLTV channels are at risk.
- **Single arbiter.** No redundancy or threshold-based arbitration.
- **Liquidity lock.** A held payment consumes HTLC slots and route liquidity, so this subtype is unsuitable for long-running delivery or arbitration windows.
- **Implementation schema required.** Deployments must publish and validate the exact service schema referenced by their kind 30361 descriptor.

### Path to cryptographic enforcement (PTLC upgrade)

A future PTLC/adaptor-signature subtype solves the enforcement problem:

- The PTLC requires the arbiter's adaptor signature to settle.
- The agent **cannot** settle unilaterally after a dispute is raised.
- The arbiter provides the adaptor to the winning party.
- This is `settlement_enforcement: cryptographic` — no trust in the agent's cooperation.

The PIP-02 event grammar and PIP-03 policy boundary may remain compatible, but the new subtype must advertise its own custody, authorization, timeout, and settlement behavior through PIP-01 and its referenced service schema.

---

## Storage schema (agent)

```sql
swaps:
  id                  TEXT PRIMARY KEY
  swap_id             TEXT UNIQUE NOT NULL
  customer_npub       TEXT NOT NULL
  agent_npub          TEXT NOT NULL
  arbiter_npub        TEXT
  amount_msat         INTEGER NOT NULL
  payment_hash        TEXT                  # SHA256(preimage)
  preimage_ciphertext BLOB                   # encrypted at rest with a data key
  bolt11_invoice      TEXT                  # private; delivered via Gift Wrap
  htlc_expiry_height  INTEGER               # actual accepted HTLC expiry height
  invoice_expires_at  INTEGER               # BOLT 11 invoice expiry time
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
  decision            TEXT                  # RELEASE | REFUND | EXPIRED
  decision_event_id   TEXT                  # Nostr event ID of CLOSED event
  created_at          INTEGER NOT NULL
  resolved_at         INTEGER
```

The arbiter maintains a separate, smaller store of swaps it is tracking and disputes it has accepted — no preimage, no payment_hash ciphertext.

---

## Service endpoints (agent)

| Method | Path                      | Description                          |
|--------|---------------------------|--------------------------------------|
| GET    | `/swap/:id`               | Get swap state                       |
| POST   | `/swap/:id/dispute`       | Raise a dispute as either party      |
| POST   | `/swap/:id/evidence`       | Submit authenticated evidence        |
| GET    | `/swap/:id/decision`      | Get arbiter decision                 |

The agent's LND is not exposed as a public endpoint. Hold-invoice operations are internal to the agent's process.

---

## Arbiter service

### Core loop

```
while running:
    ensure k30361 descriptor is published and up-to-date (if the arbiter also publishes one)
    accept Nostr events (requests, disputes)
    for each new swap request:
        record swap for potential dispute tracking
    for each dispute event:
        notify arbiter operator
        collect evidence from both parties
        evaluate against swap conditions
        if decision reached:
            publish signed CLOSED event
        if deadline approaching without decision:
            publish EXPIRED CLOSED event
```

The arbiter does **not**:
- Generate hold invoices
- Hold preimages
- Monitor HTLC states
- Settle or cancel invoices
- Custody funds

---

## Client flows

### Customer flow

1. Load Nostr keypair.
2. Connect to relays and subscribe to the PIP-02 event kinds.
3. Negotiate terms with the agent.
4. Publish the immutable `kind 7300` request and wait for the agent's acceptance transition.
5. Receive BOLT 11 hold invoice via Gift Wrap.
6. Pay the invoice via Lightning.
7. Wait for the `FUNDED` transition and validate its append-only history.
8. Confirm delivery (if goods received).
9. Wait for the `SETTLED` transition — swap complete.

If delivery never arrives:
1. Publish `DISPUTED` event before dispute window closes.
2. Submit evidence to arbiter via Gift Wrap.
3. Wait for arbiter `CLOSED` decision.
4. If decision is `REFUND` and agent honors it, HTLC cancels, funds return.

### Agent flow

1. Load Nostr keypair and Lightning node credentials (LND macaroons restricted to `invoice:write`, `invoices:read`).
2. Connect to relays.
3. Validate the customer's request and publish an acceptance transition.
4. Generate `p`, create hold invoice with CLTV >= dispute_window + buffer.
5. Deliver BOLT 11 invoice to customer via Gift Wrap. Publish `INVOICED`.
6. Wait for HTLC arrival → publish `FUNDED`.
7. If delivery confirmed: call `settle(preimage)`, publish `SETTLED`.
8. If refund condition: call `cancelinvoice`, publish `REFUNDED`.
9. If dispute raised: stop, do not settle or cancel. Wait for arbiter decision.

### Arbiter setup

1. Load nsec (env, vault, or HSM).
2. Connect to relays. Subscribe to request and dispute events.
3. No Lightning node required.
4. Run core loop.

---

## Testing

### Unit tests

- Swap request signing and validation (wrong author, expiry, missing canonical fields, invalid escrow reference).
- Transition authorization and `prev_state` coherence.
- Hold-invoice creation (verify payment_hash matches SHA256(preimage)).
- State machine transitions — reject invalid transitions (e.g., SETTLED → DISPUTED).
- Dispute event signing and validation.
- Arbiter decision event signing, actor authorization, and signature verification.
- NIP-44 encryption round-trip.
- Gift Wrap encapsulation and decapsulation.

### Integration tests

- **Regtest Lightning network.** Use Polar with two LND nodes (agent + customer wallets).
- **Local Nostr relay.** nostr-rs-relay in Docker.
- **Normal release path.** Request → acceptance → agent hold invoice → customer HTLC → FUNDED → agent settles → SETTLED.
- **Normal refund path.** Request → agent hold invoice → customer HTLC → FUNDED → agent cancels → REFUNDED.
- **Dispute release.** FUNDED → kind 7303 dispute → DISPUTED → arbiter reviews → CLOSED (RELEASE) → agent honors.
- **Dispute refund.** FUNDED → kind 7303 dispute → DISPUTED → arbiter reviews → CLOSED (REFUND) → agent honors.
- **Agent defection.** FUNDED → DISPUTED → CLOSED (REFUND) → agent ignores → HTLC expires → EXPIRED. Verify customer has signed arbiter event as evidence.
- **HTLC expiry before dispute resolution.** Request → hold invoice with short CLTV → FUNDED → dispute raised too late → HTLC expires → REFUNDED.
- **Conflicting events.** Publish transitions with incoherent `prev_state` values and verify clients reject them.
- **Snapshot disagreement.** Publish a stale or incorrect kind 30362 snapshot and verify clients prefer immutable request and transition history.

### Test harness

A scripted runner that:
1. Spawns an agent LND node and customer regtest wallet.
2. Starts a local Nostr relay.
3. Creates keypairs for customer, agent, arbiter.
4. Agent publishes the k30361 descriptor.
5. Executes each scenario and asserts: state transitions, HTLC settle/cancel outcomes, arbiter decision signatures.

---

## Security

- **nsec storage.** All keypairs encrypted at rest. Arbiters should use vaulted keys.
- **Agent's preimage storage.** `preimage_p` encrypted at rest with a vault- or HSM-protected data encryption key.
- **Lightning node access (agent only).** LND: mTLS + macaroons restricted to `invoice:write`, `invoices:read`. CLN: commando runes with minimal permissions. The arbiter does not need Lightning node access.
- **Dispute fee.** A small Lightning payment (keysend) may accompany dispute events to deter abuse; format is declared by the service schema.

## Operational

- **CLTV buffer.** Agent MUST set `htlc_expiry` >= `dispute_window` + arbiter review time + safety buffer (e.g., 1 hour).
- **Arbiter availability.** The arbiter must be online and responsive within the dispute window. SLAs should be defined.
- **Monitoring.** Alert on: FUNDED swaps approaching dispute window end without resolution, arbiter decision deadline approaching, relay disconnections.
- **Reputation.** Signed arbiter decisions are permanent public record. Malicious agent behavior (ignoring decisions) and frivolous customer disputes accumulate reputational cost.

---

## Project layout

```
pontmore/
├── README.md
├── docs/
│   ├── IMPL_GUIDE.md                       ← primary: custodial_escrow (operator-controlled)
│   └── IMPL_GUIDE_LIGHTNING_HOLD_INVOICE.md ← this file: lightning_hold_invoice (agent-controlled)
├── src/
│   ├── nostr/              ← Nostr identity, event signing, NIP-44, Gift Wrap
│   ├── swap/               ← PIP-02 request and append-only state history
│   ├── lightning/          ← agent-owned hold-invoice creation, HTLC monitoring, settle/cancel
│   └── arbiter/            ← dispute review, signed decision publishing (no custody)
└── tests/
```

---

## License

MIT
